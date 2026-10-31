```
ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow/ruby_api $ gcloud auth login
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?response_type=code&client_id=32555940559.apps.googleusercontent.com&redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&scope=openid+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fsqlservice.login+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&state=wbr4gFDB9CeIEGVWIUjzZnRwQ7HCWz&access_type=offline&code_challenge=YtlTTUMaFcd-x1IQR3p43vsFZ_1D_ekULfkQmPgwkVw&code_challenge_method=S256


You are now logged in as [anantarala@gmail.com].
Your current project is [client-portal-484417].  You can change this setting by running:
  $ gcloud config set project PROJECT_ID
ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow/ruby_api $ printf '%s' 'YOUR_SECRET' | gcloud secrets create leadflow-api-secret --data-file=-
ERROR: (gcloud.secrets.create) [anantarala@gmail.com] does not have permission to access projects instance [client-portal-484417] (or it may not exist): Permission 'secretmanager.secrets.create' denied on resource (or it may not exist). Remediate access with this Troubleshooter URL or share it with your administrator - https://console.cloud.google.com/iam-admin/troubleshooter/summary;errorId=CiQwMWExOTI2YS0xMGE2LTc1NjItODcwNi1mYjcxMDk4NzZiZGMSAA%3D%3D . This command is authenticated as anantarala@gmail.com which is the active account specified by the [core/account] property.
- '@type': type.googleapis.com/google.rpc.ErrorInfo
  domain: iam.googleapis.com
  metadata:
    error_info_id: CiQwMWExOTI2YS0xMGE2LTc1NjItODcwNi1mYjcxMDk4NzZiZGMSAA==
    permission: secretmanager.secrets.create
    troubleshooter_url: https://console.cloud.google.com/iam-admin/troubleshooter/summary;errorId=CiQwMWExOTI2YS0xMGE2LTc1NjItODcwNi1mYjcxMDk4NzZiZGMSAA%3D%3D
  reason: IAM_PERMISSION_DENIED
ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow/ruby_api $ 

```

<img width="642" height="821" alt="image" src="https://github.com/user-attachments/assets/b9ccd50d-a079-49b7-b4be-34501d8886b6" />



# LeadFlow MVP --- Complete Setup and Reference Guide

This guide records the working steps for the first LeadFlow
landscaping-business MVP: a public customer inquiry form, Ruby/Sinatra
triage API on Google Cloud Run, and Google Sheets lead storage.

## 1. What is working

Customer opens the Apps Script web-app URL → enters contact and service
details → Apps Script validates the fields → Apps Script calls the
authenticated Ruby API → Ruby classifies the inquiry as `High` or
`Normal` → Apps Script appends the lead to the `Leads` tab → customer
sees a confirmation and reference number.

Current MVP is a single demo business. Owner email notifications, spam
controls, multi-business isolation, and a business dashboard are not yet
implemented.

## 2. Project layout and values

Local Ruby project directory:

``` text
~/startup/leadflow/ruby_api/
  app.rb
  Gemfile
  Gemfile.lock
  config.ru
  Dockerfile
  .dockerignore
  .gitignore
  .env             # local only; never commit
  deploy.sh
  test/
    test_app.rb
```

Apps Script project files: - `Code.gs` - `Index.html`

Cloud settings used: - GCP project: `leadflow-513314` - Region:
`us-east1` - Cloud Run service: `leadflow-api` - API URL used:
`https://leadflow-api-917086752758.us-east1.run.app` - Health route:
`/health` - Triage route: `/api/v1/triage` - Demo business ID:
`LANDSCAPE-001`

Google spreadsheet: - Name: `LeadFlow Database` - ID used during setup:
`1Wz94TTmhz2r7gfFwOcC6LkMPX9OIF1XSqBIJAP7VyF0` - Tabs: `Leads`,
`Businesses`

`Leads` row 1 headers, in order:

``` text
lead_id | business_id | customer_name | email | phone | service | description | priority | status | created_at
```

`Businesses` row 1 headers:

``` text
business_id | business_name | owner_email
```

The Apps Script uses `SpreadsheetApp.getActiveSpreadsheet()`, so bind it
to the intended spreadsheet.

## 3. Create and verify the spreadsheet

Create the spreadsheet and the two tabs above. In Apps Script, this
optional function verifies access:

``` javascript
function testLeadFlowConnection() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  Logger.log("Spreadsheet: " + ss.getName());
  Logger.log("Spreadsheet ID: " + ss.getId());
  Logger.log("Sheets: " + ss.getSheets().map(s => s.getName()).join(", "));
}
```

Run `testLeadFlowConnection` and approve permissions. The execution log
should show `LeadFlow Database` and the two tab names.

## 4. Ruby/Sinatra API

Create the local directory and enter it:

``` bash
mkdir -p ~/startup/leadflow/ruby_api
cd ~/startup/leadflow/ruby_api
```

### `Gemfile`

``` ruby
source "https://rubygems.org"

gem "sinatra", "~> 4.2"
gem "puma", "~> 6.6"
gem "dotenv"
```

Install gems and create/update the lockfile:

``` bash
bundle install
```

Commit `Gemfile.lock` to source control so builds use locked dependency
versions.

### `app.rb`

``` ruby
require 'sinatra/base'
require 'json'
require 'dotenv/load'

class LeadFlowAPI < Sinatra::Base
  configure do
    set :show_exceptions, false
    set :raise_errors, false
    set :bind, '0.0.0.0'
    set :port, ENV.fetch('PORT', 4567).to_i

    set :host_authorization, permitted_hosts: [
      'leadflow-api-917086752758.us-east1.run.app',
      'localhost',
      '127.0.0.1'
    ]
  end

  before do
    content_type :json
  end

  get '/health' do
    {
      status: 'ok',
      service: 'leadflow-ruby-api'
    }.to_json
  end

  post '/api/v1/triage' do
    authenticate!
    data = JSON.parse(request.body.read)

    name = clean(data['name'], 100)
    service = clean(data['service'], 80)
    description = clean(data['description'], 1500)

    if name.empty? || service.empty?
      halt 422, { error: 'name and service are required' }.to_json
    end

    text = "#{service} #{description}".downcase
    priority = text.match?(/\b(urgent|today|tomorrow|storm damage|fallen tree|hazard|unsafe)\b/) ? 'High' : 'Normal'

    summary = "#{service} inquiry from #{name}. #{description.empty? ? 'No additional details provided.' : description[0, 180]} Staff review required."

    {
      priority: priority,
      summary: summary
    }.to_json
  rescue JSON::ParserError
    halt 400, { error: 'invalid JSON' }.to_json
  end

  private

  def authenticate!
    expected = ENV.fetch('LEADFLOW_API_SECRET', '')
    supplied = request.env['HTTP_AUTHORIZATION'].to_s.sub(/\ABearer\s+/i, '')

    valid = !expected.empty? &&
            supplied.bytesize == expected.bytesize &&
            Rack::Utils.secure_compare(supplied, expected)

    halt 401, { error: 'unauthorized' }.to_json unless valid
  end

  def clean(value, max)
    value.to_s.gsub(/[<>]/, '').strip[0, max]
  end
end

if $PROGRAM_NAME == __FILE__
  LeadFlowAPI.run!
end
```

**Why the configuration matters:** Cloud Run must reach the process over
`0.0.0.0`, not only localhost. `PORT` is supplied by Cloud Run (8080).
Sinatra 4 host authorization must explicitly allow the deployed hostname
and local development hosts. If the Cloud Run URL changes, update the
permitted hostname and redeploy.

### `.env`

Create this file in `ruby_api/`:

``` dotenv
LEADFLOW_API_SECRET=REPLACE_WITH_A_LONG_RANDOM_SECRET
```

Replace the placeholder locally with a long random secret. Do not commit
`.env`, paste the secret into chat, or put it in `Code.gs` /
`Index.html`.

### Local tests

Start the server:

``` bash
cd ~/startup/leadflow/ruby_api
bundle exec ruby app.rb
```

In another terminal:

``` bash
curl http://localhost:8080/health
```

Expected:

``` json
{"status":"ok","service":"leadflow-ruby-api"}
```

Stop the server with `Ctrl+C`.

Test the protected triage endpoint (with `.env` in the current
directory):

``` bash
set -a
source .env
set +a

curl -X POST \
  http://localhost:8080/api/v1/triage \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LEADFLOW_API_SECRET}" \
  -d '{
    "name": "John Smith",
    "service": "Tree Removal",
    "description": "A large tree fell during the storm and is blocking my driveway. Need help today."
  }'

unset LEADFLOW_API_SECRET
```

Expected priority: `High`. A missing or incorrect bearer token should
return HTTP `401`.

## 5. Docker files

### `Dockerfile`

The initial build failed compiling native gem `nio4r` because compiler
tools were missing. This Dockerfile adds `build-essential`:

``` dockerfile
FROM ruby:3.3-slim

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends build-essential && \
    rm -rf /var/lib/apt/lists/*

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

ENV PORT=8080
EXPOSE 8080

CMD ["bundle", "exec", "ruby", "app.rb"]
```

### `.dockerignore`

``` text
.env
.env.*
!.env.example
.git
.gitignore
.bundle
vendor/bundle
log
tmp
test
```

### `.gitignore`

``` text
.env
.env.*
!.env.example
.bundle
vendor/bundle
log
tmp
```

The deployment script can be committed. `.env` cannot. Check before
committing:

``` bash
git status
git check-ignore -v .env
```

## 6. Google Cloud setup

Authenticate and select the project:

``` bash
gcloud auth login
gcloud config set project leadflow-513314
gcloud config get-value project
```

Enable required services:

``` bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  --project=leadflow-513314
```

If billing is not linked, list accounts:

``` bash
gcloud billing accounts list
```

Then link the intended billing account, using its actual ID:

``` bash
gcloud billing projects link leadflow-513314 \
  --billing-account=YOUR_BILLING_ACCOUNT_ID
```

Billing is required for deploying billable Cloud services. Check the
billing dashboard. Cloud Run minimum instances are set to zero below,
but builds, requests, storage, and networking can still incur charges.

## 7. Deployment script

Create `deploy.sh` in `ruby_api/`:

``` bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE_NAME="leadflow-api"
REGION="us-east1"
PROJECT_ID="leadflow-513314"

if [[ ! -f ".env" ]]; then
  echo "ERROR: .env file not found."
  exit 1
fi

if [[ ! -f "Dockerfile" || ! -f "Gemfile.lock" ]]; then
  echo "ERROR: Dockerfile or Gemfile.lock is missing."
  exit 1
fi

set -a
source .env
set +a

if [[ -z "${LEADFLOW_API_SECRET:-}" ]]; then
  echo "ERROR: LEADFLOW_API_SECRET is missing from .env."
  exit 1
fi

gcloud config set project "$PROJECT_ID"

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region "$REGION" \
  --project "$PROJECT_ID" \
  --port 8080 \
  --min 0 \
  --max 3 \
  --set-env-vars="LEADFLOW_API_SECRET=${LEADFLOW_API_SECRET}" \
  --allow-unauthenticated \
  --quiet

unset LEADFLOW_API_SECRET

echo
echo "Deployment complete."
gcloud run services describe "$SERVICE_NAME" \
  --region "$REGION" \
  --project "$PROJECT_ID" \
  --format="value(status.url)"
```

Make executable and deploy:

``` bash
chmod +x deploy.sh
./deploy.sh
```

This MVP passes the secret as a Cloud Run environment variable on
deployment. Do not echo it or commit `.env`. A secret-management service
would be stronger for production, but the initial build used this
simpler approach to keep the MVP inexpensive.

## 8. Verify Cloud Run

Get the service URL:

``` bash
gcloud run services describe leadflow-api \
  --region us-east1 \
  --project leadflow-513314 \
  --format="value(status.url)"
```

Health test:

``` bash
curl https://leadflow-api-917086752758.us-east1.run.app/health
```

Expected:

``` json
{"status":"ok","service":"leadflow-ruby-api"}
```

Live triage test:

``` bash
cd ~/startup/leadflow/ruby_api
set -a
source .env
set +a

curl -X POST \
  https://leadflow-api-917086752758.us-east1.run.app/api/v1/triage \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LEADFLOW_API_SECRET}" \
  -d '{
    "name": "John Smith",
    "service": "Tree Removal",
    "description": "A large tree fell during the storm and is blocking my driveway. Need help today."
  }'

unset LEADFLOW_API_SECRET
```

Expected: JSON containing `"priority":"High"` and a summary.

### Read logs if deployment fails

``` bash
gcloud run services logs read leadflow-api \
  --region us-east1 \
  --project leadflow-513314 \
  --limit 100
```

For one revision:

``` bash
gcloud logging read \
'resource.type="cloud_run_revision"
 AND resource.labels.service_name="leadflow-api"
 AND resource.labels.revision_name="YOUR_REVISION_NAME"' \
--project=leadflow-513314 \
--limit=50 \
--format="table(timestamp,severity,textPayload)"
```

Issues encountered and fixes: - Native `nio4r` compile failed: add
`build-essential` in Dockerfile. - Cloud Run could not reach container:
bind Sinatra to `0.0.0.0` and use Cloud Run's `PORT`. -
`Host not permitted`: explicitly permit the Cloud Run hostname using
Sinatra's `host_authorization` setting.

## 9. Apps Script configuration

In the Apps Script editor, open **Project Settings → Script Properties →
Add script property**:

  Property                Value
  ----------------------- -------------------------------------
  `LEADFLOW_API_SECRET`   Same secret configured in Cloud Run

Keep the secret in Script Properties, not in browser HTML or source
code.

### `Code.gs` --- lead triage and Sheets write

``` javascript
function processLead(lead) {
  const properties = PropertiesService.getScriptProperties();
  const secret = properties.getProperty('LEADFLOW_API_SECRET');

  if (!secret) {
    throw new Error('Missing API secret');
  }

  const url = 'https://leadflow-api-917086752758.us-east1.run.app/api/v1/triage';

  const response = UrlFetchApp.fetch(url, {
    method: 'post',
    contentType: 'application/json',
    headers: {
      Authorization: 'Bearer ' + secret
    },
    payload: JSON.stringify({
      name: lead.name,
      service: lead.service,
      description: lead.description
    }),
    muteHttpExceptions: true
  });

  if (response.getResponseCode() !== 200) {
    throw new Error('Triage API failed: ' + response.getContentText());
  }

  const result = JSON.parse(response.getContentText());

  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('Leads');

  if (!sheet) {
    throw new Error('Leads sheet not found');
  }

  const leadId = 'LF-' + Utilities.getUuid().slice(0, 8).toUpperCase();
  const createdAt = new Date();

  sheet.appendRow([
    leadId,
    lead.business_id || '',
    lead.name,
    lead.email || '',
    lead.phone || '',
    lead.service,
    lead.description || '',
    result.priority,
    'New',
    createdAt
  ]);

  Logger.log('Lead saved: ' + leadId);
  Logger.log('Priority: ' + result.priority);

  return {
    lead_id: leadId,
    priority: result.priority,
    summary: result.summary
  };
}
```

### Test Apps Script → Ruby

``` javascript
function testRubyTriage() {
  const properties = PropertiesService.getScriptProperties();
  const secret = properties.getProperty('LEADFLOW_API_SECRET');

  if (!secret) {
    throw new Error('LEADFLOW_API_SECRET is missing from Script Properties');
  }

  const url = 'https://leadflow-api-917086752758.us-east1.run.app/api/v1/triage';

  const payload = {
    name: 'John Smith',
    service: 'Tree Removal',
    description: 'A large tree fell during the storm and is blocking my driveway. Need help today.'
  };

  const response = UrlFetchApp.fetch(url, {
    method: 'post',
    contentType: 'application/json',
    headers: {
      Authorization: 'Bearer ' + secret
    },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });

  Logger.log('HTTP Status: ' + response.getResponseCode());
  Logger.log('Ruby API Response: ' + response.getContentText());

  if (response.getResponseCode() !== 200) {
    throw new Error('Ruby API request failed');
  }

  return JSON.parse(response.getContentText());
}
```

### Test Apps Script → Ruby → Sheets

``` javascript
function testProcessLead() {
  const result = processLead({
    business_id: 'LANDSCAPE-001',
    name: 'John Smith',
    email: 'john@example.com',
    phone: '555-123-4567',
    service: 'Tree Removal',
    description: 'A large tree fell during the storm and is blocking my driveway. Need help today.'
  });

  Logger.log(JSON.stringify(result));
}
```

Run `testRubyTriage` to verify HTTP 200. Run `testProcessLead` once to
append a test row. Every run creates a new row; there is no duplicate
detection yet.

## 10. Public customer inquiry form

Apps Script serves the form. Browser JavaScript calls the server-side
`submitLead()` function through `google.script.run`; the API secret is
never placed in the HTML.

### Add to `Code.gs`

Keep `processLead()` and add:

``` javascript
function doGet() {
  return HtmlService
    .createHtmlOutputFromFile('Index')
    .setTitle('Request a Landscaping Quote')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}

function submitLead(formData) {
  const name = String(formData.name || '').trim();
  const email = String(formData.email || '').trim();
  const phone = String(formData.phone || '').trim();
  const service = String(formData.service || '').trim();
  const description = String(formData.description || '').trim();

  if (!name || !email || !phone || !service || !description) {
    throw new Error('Please complete all fields.');
  }

  if (name.length > 100 || email.length > 254 ||
      phone.length > 30 || service.length > 80 ||
      description.length > 1500) {
    throw new Error('One or more fields are too long.');
  }

  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    throw new Error('Please enter a valid email address.');
  }

  return processLead({
    business_id: 'LANDSCAPE-001',
    name: name,
    email: email,
    phone: phone,
    service: service,
    description: description
  });
}
```

### `Index.html`

In Apps Script, add an HTML file named `Index` (or replace the existing
`Index.html`) with:

``` html
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <style>
    * { box-sizing: border-box; }

    body {
      margin: 0;
      padding: 30px 16px;
      background: #f4f7f4;
      color: #243329;
      font-family: Arial, sans-serif;
    }

    .container {
      max-width: 560px;
      margin: 20px auto;
      padding: 30px;
      background: white;
      border-radius: 12px;
      box-shadow: 0 4px 18px rgba(0, 0, 0, 0.07);
    }

    h1 {
      margin-top: 0;
      color: #245c38;
      font-size: 27px;
    }

    .subtitle {
      color: #66736a;
      line-height: 1.5;
      margin-bottom: 25px;
    }

    label {
      display: block;
      margin: 16px 0 7px;
      font-weight: 600;
      font-size: 14px;
    }

    input, select, textarea {
      width: 100%;
      padding: 12px;
      border: 1px solid #ccd5ce;
      border-radius: 7px;
      font-size: 15px;
      background: white;
    }

    textarea {
      min-height: 110px;
      resize: vertical;
    }

    input:focus, select:focus, textarea:focus {
      outline: 2px solid #77a987;
      border-color: #245c38;
    }

    button {
      width: 100%;
      margin-top: 24px;
      padding: 14px;
      border: none;
      border-radius: 7px;
      background: #245c38;
      color: white;
      font-size: 16px;
      font-weight: bold;
      cursor: pointer;
    }

    button:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    #message {
      display: none;
      margin-top: 18px;
      padding: 14px;
      border-radius: 7px;
      line-height: 1.5;
      overflow-wrap: anywhere;
    }

    .success {
      display: block !important;
      background: #e8f5eb;
      color: #205b32;
    }

    .error {
      display: block !important;
      background: #fdecec;
      color: #9b2525;
    }

    .footer {
      margin-top: 22px;
      text-align: center;
      color: #89928b;
      font-size: 12px;
    }
  </style>
</head>

<body>
  <main class="container">
    <h1>Request a Landscaping Quote</h1>

    <p class="subtitle">
      Tell us what you need. Our team will review your request
      and get back to you.
    </p>

    <form id="leadForm">
      <label for="name">Full name *</label>
      <input id="name" name="name" maxlength="100"
             autocomplete="name" required>

      <label for="email">Email address *</label>
      <input id="email" name="email" type="email"
             maxlength="254" autocomplete="email" required>

      <label for="phone">Phone number *</label>
      <input id="phone" name="phone" type="tel"
             maxlength="30" autocomplete="tel" required>

      <label for="service">Service needed *</label>
      <select id="service" name="service" required>
        <option value="">Select a service</option>
        <option value="Lawn Care">Lawn Care</option>
        <option value="Landscaping">Landscaping</option>
        <option value="Tree Removal">Tree Removal</option>
        <option value="Snow Removal">Snow Removal</option>
        <option value="Other">Other</option>
      </select>

      <label for="description">Describe the work needed *</label>
      <textarea id="description" name="description"
                maxlength="1500"
                placeholder="Tell us about your project..."
                required></textarea>

      <button id="submitButton" type="submit">
        Submit Request
      </button>
    </form>

    <div id="message" role="status" aria-live="polite"></div>

    <div class="footer">
      Powered by LeadFlow
    </div>
  </main>

  <script>
    const form = document.getElementById('leadForm');
    const button = document.getElementById('submitButton');
    const message = document.getElementById('message');

    form.addEventListener('submit', function(event) {
      event.preventDefault();

      if (!form.reportValidity()) {
        return;
      }

      const formData = {
        name: form.elements.name.value.trim(),
        email: form.elements.email.value.trim(),
        phone: form.elements.phone.value.trim(),
        service: form.elements.service.value,
        description: form.elements.description.value.trim()
      };

      button.disabled = true;
      button.textContent = 'Submitting...';
      message.className = '';
      message.style.display = 'none';

      google.script.run
        .withSuccessHandler(function(result) {
          message.className = 'success';
          message.textContent =
            'Thank you! Your request has been received. ' +
            'Your reference number is ' + result.lead_id + '.';

          form.reset();
          button.disabled = false;
          button.textContent = 'Submit Request';
        })
        .withFailureHandler(function(error) {
          message.className = 'error';
          message.textContent =
            error && error.message
              ? error.message
              : 'Something went wrong. Please try again.';

          button.disabled = false;
          button.textContent = 'Submit Request';
        })
        .submitLead(formData);
    });
  </script>
</body>
</html>
```

### Deploy the Apps Script web app

1.  Save the Apps Script project.
2.  Choose **Deploy → New deployment**.
3.  Select the gear icon and choose **Web app**.
4.  Set **Execute as** to `Me`.
5.  Set **Who has access** to `Anyone` for the public demo.
6.  Click **Deploy** and complete authorization.
7.  Copy the Web app URL.
8.  Open it in an incognito/private window and submit a test lead.

For subsequent code changes, use **Deploy → Manage deployments → Edit →
New version → Deploy**. Saving the editor does not automatically update
an existing deployed version.

Test values:

``` json
{
  "name": "John Smith",
  "email": "john@example.com",
  "phone": "555-123-4567",
  "service": "Tree Removal",
  "description": "A tree fell during the storm and is blocking my driveway."
}
```

The user confirmed the form submission worked and created a lead. Note:
current Ruby priority logic is keyword-based, not AI. It marks `High`
when the combined service/description includes `urgent`, `today`,
`tomorrow`, `storm damage`, `fallen tree`, `hazard`, or `unsafe`. The
sample description "A tree fell during the storm..." may not match the
exact `storm damage` or `fallen tree` phrases and could be `Normal`.

## 11. Git safety and commands

From project root:

``` bash
cd ~/startup/leadflow
git init
git add .
git status
```

Ensure `.env` is not staged. If accidentally staged:

``` bash
git restore --staged ruby_api/.env
git check-ignore -v ruby_api/.env
```

Commit:

``` bash
git add .
git commit -m "Build LeadFlow landscaping lead intake MVP"
```

Check remotes and push if configured:

``` bash
git remote -v
git push -u origin main
```

Use the actual branch name if it is not `main`. `deploy.sh` can be
committed; `.env` and real customer data must not be committed.

## 12. Quick command reference

Start local API:

``` bash
cd ~/startup/leadflow/ruby_api
bundle install
bundle exec ruby app.rb
```

Local health check:

``` bash
curl http://localhost:8080/health
```

Live health check:

``` bash
curl https://leadflow-api-917086752758.us-east1.run.app/health
```

Deploy API:

``` bash
cd ~/startup/leadflow/ruby_api
./deploy.sh
```

Get Cloud Run URL:

``` bash
gcloud run services describe leadflow-api \
  --region us-east1 \
  --project leadflow-513314 \
  --format="value(status.url)"
```

Read service logs:

``` bash
gcloud run services logs read leadflow-api \
  --region us-east1 \
  --project leadflow-513314 \
  --limit 100
```

Git status:

``` bash
cd ~/startup/leadflow
git status
```

## 13. Known limitations and next steps

The working MVP is a single-business demo using `LANDSCAPE-001`. Before
selling broadly:

1.  Add automatic email notification to the business owner.
2.  Add spam protection and submission rate limiting; the public form
    currently has neither.
3.  Add duplicate-submission protection.
4.  Make business ID, branding, owner email, and available services
    configurable per customer.
5.  Implement business authentication and strict per-business data
    isolation. The current app has no owner login/dashboard.
6.  Add a privacy notice and a customer-data retention/deletion policy.
7.  Add monitoring, backups, and a way to review failed submissions.
8.  Review security and secret handling before processing real customer
    data at scale.

The current priority classification is a simple keyword rule, not AI.
Describe it accurately as automatic priority tagging with staff review
required. The Ruby response summary also says `Staff review required.`

## 14. Completion checklist

-   [x] Create spreadsheet and `Leads` / `Businesses` tabs.
-   [x] Build Ruby/Sinatra API.
-   [x] Add bearer-token authentication.
-   [x] Test API locally.
-   [x] Build Docker image with native build tools.
-   [x] Bind app to `0.0.0.0` and Cloud Run `PORT`.
-   [x] Configure Sinatra host authorization.
-   [x] Deploy API to Cloud Run.
-   [x] Verify live health endpoint.
-   [x] Verify authenticated live triage endpoint.
-   [x] Store API secret in Apps Script Script Properties.
-   [x] Connect Apps Script to Ruby.
-   [x] Save classified leads to Google Sheets.
-   [x] Build and deploy public customer form.
-   [x] Test public form successfully.
-   [ ] Add owner email notifications.
-   [ ] Add spam and rate-limit protection.
-   [ ] Support multiple businesses safely.
-   [ ] Add business-facing lead dashboard.
-   [ ] Complete production security and privacy review.

**Secret reminder:** Never put the actual `LEADFLOW_API_SECRET` in this
document, GitHub, `Index.html`, or a message. The `.env` file is
local-only.
