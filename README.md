# Ameer-Hamza
Hi guys please star my channel
require 'sinatra/base'  # Use the Sinatra web framework
require 'octokit'       # Use the Octokit Ruby library to interact with GitHub's REST API
require 'dotenv/load'   # Manages environment variables
require 'json'          # Allows your app to manipulate JSON data
require 'openssl'       # Verifies the webhook signature
require 'jwt'           # Authenticates a GitHub App
require 'time'          # Gets ISO 8601 representation of a Time object
require 'logger'        # Logs debug statements

# This code is a Sinatra app, for two reasons:
#   1. Because the app will require a landing page for installation.
#   2. To easily handle webhook events.

class GHAapp < Sinatra::Application

  # Sets the port that's used when starting the web server.
  set :port, 3000
  set :bind, '0.0.0.0'

  # Expects the private key in PEM format. Converts the newlines.
  PRIVATE_KEY = OpenSSL::PKey::RSA.new(ENV['GITHUB_PRIVATE_KEY'].gsub('\n', "\n"))

  # Your registered app must have a webhook secret.
  # The secret is used to verify that webhooks are sent by GitHub.
  WEBHOOK_SECRET = ENV['GITHUB_WEBHOOK_SECRET']

  # The GitHub App's identifier (type integer).
  APP_IDENTIFIER = ENV['GITHUB_APP_IDENTIFIER']

  # Turn on Sinatra's verbose logging during development
  configure :development do
    set :logging, Logger::DEBUG
  end

  # Executed before each request to the `/event_handler` route
  before '/event_handler' do
    get_payload_request(request)
    verify_webhook_signature

    # If a repository name is provided in the webhook, validate that
    # it consists only of latin alphabetic characters, `-`, and `_`.
    unless @payload['repository'].nil?
      halt 400 if (@payload['repository']['name'] =~ /[0-9A-Za-z\-\_]+/).nil?
    end

    authenticate_app
    # Authenticate the app installation in order to run API operations
    authenticate_installation(@payload)
  end

  post '/event_handler' do

    # Get the event type from the HTTP_X_GITHUB_EVENT header
    case request.env['HTTP_X_GITHUB_EVENT']

    when 'check_suite'
      # A new check_suite has been created. Create a new check run with status queued
      if @payload['action'] == 'requested' || @payload['action'] == 'rerequested'
        create_check_run
      end

    when 'check_run'
      # Check that the event is being sent to this app
      if @payload['check_run']['app']['id'].to_s === APP_IDENTIFIER
        case @payload['action']
        when 'created'
          initiate_check_run
        when 'rerequested'
          create_check_run
        when 'requested_action'
          take_requested_action
        end
      end
    end

    200 # success status
  end

  helpers do

    # Create a new check run with status "queued"
    def create_check_run
      @installation_client.create_check_run(
        # [String, Integer, Hash, Octokit Repository object] A GitHub repository.
        @payload['repository']['full_name'],
        # [String] The name of your check run.
        'Octo RuboCop',
        # [String] The SHA of the commit to check
        # The payload structure differs depending on whether a check run or a check suite event occurred.
        @payload['check_run'].nil? ? @payload['check_suite']['head_sha'] : @payload['check_run']['head_sha'],
        # [Hash] 'Accept' header option, to avoid a warning about the API not being ready for production use.
        accept: 'application/vnd.github+json'
      )
    end

    # Start the CI process
    def initiate_check_run
      # Once the check run is created, you'll update the status of the check run
      # to 'in_progress' and run the CI process. When the CI finishes, you'll
      # update the check run status to 'completed' and add the CI results.

      @installation_client.update_check_run(
        @payload['repository']['full_name'],
        @payload['check_run']['id'],
        status: 'in_progress',
        accept: 'application/vnd.github+json'
      )

      full_repo_name = @payload['repository']['full_name']
      repository     = @payload['repository']['name']
      head_sha       = @payload['check_run']['head_sha']

      clone_repository(full_repo_name, repository, head_sha)

      # Run RuboCop on all files in the repository
      @report = `rubocop '#{repository}' --format json`
      logger.debug @report
      `rm -rf #{repository}`
      @output = JSON.parse @report

      annotations = []
      # You can create a maximum of 50 annotations per request to the Checks
      # API. To add more than 50 annotations, use the "Update a check run" API
      # endpoint. This example code limits the number of annotations to 50.
      # See /rest/reference/checks#update-a-check-run
      # for details.
      max_annotations = 50

      # RuboCop reports the number of errors found in "offense_count"
      if @output['summary']['offense_count'] == 0
        conclusion = 'success'
      else
        conclusion = 'neutral'
        @output['files'].each do |file|

          # Only parse offenses for files in this app's repository
          file_path = file['path'].gsub(/#{repository}\//,'')
          annotation_level = 'notice'

          # Parse each offense to get details and location
          file['offenses'].each do |offense|
            # Limit the number of annotations to 50
            next if max_annotations == 0
            max_annotations -= 1

            start_line   = offense['location']['start_line']
            end_line     = offense['location']['last_line']
            start_column = offense['location']['start_column']
            end_column   = offense['location']['last_column']
            message      = offense['message']

            # Create a new annotation for each error
            annotation = {
              path: file_path,
              start_line: start_line,
              end_line: end_line,
              start_column: start_column,
              end_column: end_column,
              annotation_level: annotation_level,
              message: message
            }
            # Annotations only support start and end columns on the same line
            if start_line == end_line
              annotation.merge({start_column: start_column, end_column: end_column})
            end

            annotations.push(annotation)
          end
        end
      end

      # Updated check run summary and text parameters
      summary = "Octo RuboCop summary\n-Offense count: #{@output['summary']['offense_count']}\n-File count: #{@output['summary']['target_file_count']}\n-Target file count: #{@output['summary']['inspected_file_count']}"
      text = "Octo RuboCop version: #{@output['metadata']['rubocop_version']}"

      # Mark the check run as complete! And if there are warnings, share them.
      @installation_client.update_check_run(
        @payload['repository']['full_name'],
        @payload['check_run']['id'],
        status: 'completed',
        conclusion: conclusion,
        output: {
          title: 'Octo RuboCop',
          summary: summary,
          text: text,
          annotations: annotations
        },
        actions: [{
          label: 'Fix this',
          description: 'Automatically fix all linter notices.',
          identifier: 'fix_rubocop_notices'
        }],
        accept: 'application/vnd.github+json'
      )
    end

    # Clones the repository to the current working directory, updates the
    # contents using Git pull, and checks out the ref.
    #
    # full_repo_name  - The owner and repo. Ex: octocat/hello-world
    # repository      - The repository name
    # ref             - The branch, commit SHA, or tag to check out
    def clone_repository(full_repo_name, repository, ref)
      @git = Git.clone("https://x-access-token:#{@installation_token.to_s}@github.com/#{full_repo_name}.git", repository)
      pwd = Dir.getwd()
      Dir.chdir(repository)
      @git.pull
      @git.checkout(ref)
      Dir.chdir(pwd)
    end

    # Handles the check run `requested_action` event
    # See /webhooks/event-payloads/#check_run
    def take_requested_action
      full_repo_name = @payload['repository']['full_name']
      repository     = @payload['repository']['name']
      head_branch    = @payload['check_run']['check_suite']['head_branch']

      if (@payload['requested_action']['identifier'] == 'fix_rubocop_notices')
        clone_repository(full_repo_name, repository, head_branch)

        # Sets your commit username and email address
        @git.config('user.name', ENV['GITHUB_APP_USER_NAME'])
        @git.config('user.email', ENV['GITHUB_APP_USER_EMAIL'])

        # Automatically correct RuboCop style errors
        @report = `rubocop '#{repository}/*' --format json --auto-correct`

        pwd = Dir.getwd()
        Dir.chdir(repository)
        begin
          @git.commit_all('Automatically fix Octo RuboCop notices.')
          @git.push("https://x-access-token:#{@installation_token.to_s}@github.com/#{full_repo_name}.git", head_branch)
        rescue
          # Nothing to commit!
          puts 'Nothing to commit'
        end
        Dir.chdir(pwd)
        `rm -rf '#{repository}'`
      end
    end

    # Saves the raw payload and converts the payload to JSON format
    def get_payload_request(request)
      # request.body is an IO or StringIO object
      # Rewind in case someone already read it
      request.body.rewind
      # The raw text of the body is required for webhook signature verification
      @payload_raw = request.body.read
      begin
        @payload = JSON.parse @payload_raw
      rescue => e
        fail 'Invalid JSON (#{e}): #{@payload_raw}'
      end
    end

    # Instantiate an Octokit client authenticated as a GitHub App.
    # GitHub App authentication requires that you construct a
    # JWT (https://jwt.io/introduction/) signed with the app's private key,
    # so GitHub can be sure that it came from the app and not altered by
    # a malicious third party.
    def authenticate_app
      payload = {
          # The time that this JWT was issued, _i.e._ now.
          iat: Time.now.to_i,

          # JWT expiration time (10 minute maximum)
          exp: Time.now.to_i + (10 * 60),

          # Your GitHub App's identifier number
          iss: APP_IDENTIFIER
      }

      # Cryptographically sign the JWT.
      jwt = JWT.encode(payload, PRIVATE_KEY, 'RS256')

      # Create the Octokit client, using the JWT as the auth token.
      @app_client ||= Octokit::Client.new(bearer_token: jwt)
    end

    # Instantiate an Octokit client, authenticated as an installation of a
    # GitHub App, to run API operations.
    def authenticate_installation(payload)
      @installation_id = payload['installation']['id']
      @installation_token = @app_client.create_app_installation_access_token(@installation_id)[:token]
      @installation_client = Octokit::Client.new(bearer_token: @installation_token)
    end

    # Check X-Hub-Signature to confirm that this webhook was generated by
    # GitHub, and not a malicious third party.
    #
    # GitHub uses the WEBHOOK_SECRET, registered to the GitHub App, to
    # create the hash signature sent in the `X-HUB-Signature` header of each
    # webhook. This code computes the expected hash signature and compares it to
    # the signature sent in the `X-HUB-Signature` header. If they don't match,
    # this request is an attack, and you should reject it. GitHub uses the HMAC
    # hexdigest to compute the signature. The `X-HUB-Signature` looks something
    # like this: 'sha1=123456'.
    def verify_webhook_signature
      their_signature_header = request.env['HTTP_X_HUB_SIGNATURE'] || 'sha1='
      method, their_digest = their_signature_header.split('=')
      our_digest = OpenSSL::HMAC.hexdigest(method, WEBHOOK_SECRET, @payload_raw)
      halt 401 unless their_digest == our_digest

      # The X-GITHUB-EVENT header provides the name of the event.
      # The action value indicates the which action triggered the event.
      logger.debug "---- received event #{request.env['HTTP_X_GITHUB_EVENT']}"
      logger.debug "----    action #{@payload['action']}" unless @payload['action'].nil?
    end

  end

  # Finally some logic to let us run this server directly from the command line,
  # or with Rack. Don't worry too much about this code. But, for the curious:
  # $0 is the executed file
  # __FILE__ is the current file
  # If they are the same—that is, we are running this file directly, call the
  # Sinatra run method
  run! if __FILE__ == $0
end
