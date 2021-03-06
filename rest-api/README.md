# The RDR API

## Cloud projects for testing

This api is implemented using Flask. They are served by a AppEngine instance.

The App Engine project used for testing is pmi-drc-api-test, billed to Vanderbilt.

The GCS bucket for biobank manifest testing is pmi-drc-biobank-test.

## Configuring a Ubuntu workstation for API development:

Follow the instructions in the [client directory](../rdr_client) first to set up
a virtual Python environment, then using the client venv, follow the
instructions here.

### Installing dependencies

Make sure that you have google
[cloud SDK](https://cloud.google.com/sdk/downloads) installed.

From the rest-api directory, run:

* `sudo apt-get install mysql-server libmysqlclient-dev` (to install MySQL server and client —
if you prefer to use docker, see note below)
* `tools/setup_env.sh` (get libs, set up git hooks)
* `mysql -V` to ensure that you have mysql >= 5.7
* `dev_appserver.py test.yaml --require_indexes` (to run your local server)
* `tools/setup_local_database.sh` (to create a database in MySQL, upgrade its schema,
  import the latest codebook into it, and put the config for the database in Datastore)


#### If you prefer to run mysql via docker

```
docker volume create rdr-mysql
docker run --rm --name pmi-rdr-mysql -d \
  -p 3306:3306 \
  --volume rdr-mysql:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  mysql:5.7
```


### Running the development app server

From the rest-api directory, you can in general run your local server with:

```Shell
dev_appserver.py test.yaml --require_indexes &
```

This runs a local server with both API and offline services (suitable for local
development as well as running client tests).

### Configuring your instance

When the instance comes up for the first time, it will have no configuration, and be generally useless.

The best way to get set up for development is to install the dev config:

```Shell
tools/install_config.sh --config config/config_dev.json --update
```
The server should be now be good to go!

In order to modify the configuration manually:

(Note: For local development we need an `user_info` map entry with a key of
`example@example.com`.  This is what a oauth user appears as under the
dev_appserver.)


If running a local dev_appserver, navigate to the
[datastore viewer](http://localhost:8000/datastore?kind=Config).
You should be able to modify config settings using the fancy UI.

If running in production, go to the
[cloud console](https://console.cloud.google.com).  Select the app engine
project and then click on "datastore" in the left hand navigation bar.

### Running the tests against the local appserver

Start the dev appserver, then from the rest-api directory run:

```Shell
test/run_tests.sh -g $sdk_dir
```

This will run both the unit tests and the client tests.

If you want to be super slick, and have the tests run every time you change a
source file, you can do this.

(You will have to install ack-grep and entr if you haven't already.)

```Shell
until ack-grep -f --python | entr -r test/run_tests.sh -g $sdk_dir unit;do sleep 1; done
```

### Adding fake participants to local appserver

See [rdr_client/README.md](../rdr_client/README.md) for instructions.

Your `config_dev.json` loaded earlier should include a Config entity with
`config_key=allow_nonprod_requests` and `value=True`. You can check the
current config by running `tools/install_config.sh` with no arguments.

## Deploying to test server

### Before uploading a new version

Update the database schema. Schema updates should be backwards-compatible to
facilitate rollbacks. (If you are replacing (and not updating) the schema,
run
`tools/connect_to_database.sh --project pmi-drc-api-test --account $USER@google.com`
and `DROP DATABASE rdr; CREATE DATABASE rdr;`.)
`tools/upgrade_database.sh --project pmi-drc-api-test --account $USER@google.com`
will upgrade the test project's database. TODO(DA-211) automate this.

If you are adding a new enum or custom type referenced by fields to the schema,
you will need to update script.py.mako to import them. (The imports listed there
are used in our generated Alembic versions.)

### Deploying

To deploy to the test server, `https://pmi-drc-api-test.appspot.com/`, first get your
Git repo into the desired state, then run the following from the rest-api directory:

```Shell
tools/deploy_app.sh --version HEAD --deploy_as_version $USER_my_feature \
    --account $USER@google.com --project pmi-drc-api-test
```

By default this creates an automatically named version like `20170314t094223`
and promotes it to receive all traffic / be the default serving version.

If you've changed other files you may need to deploy them as well, for instance the cron config:
```Shell
gcloud app deploy cron.yaml
```

### Running client tests against test server

From the rest-api directory, run:

```Shell
test/test_server.sh
```

This will execute all the client tests in turn against the test server, stopping
if any one of them fails. To run a particular test use the -r flag as for
run_tests.

## Auth Model

The RDR has separate permissions management for gcloud project administration,
RDR's custom config updates, and general API endpoints.

### Cloud Project Admin Permissions

[Cloud Platform Admin settings](https://console.cloud.google.com/iam-admin/serviceaccounts/project?project=all-of-us-rdr-staging)
control which people can administer the project. Service accounts (and their
keys) are also created here.

Admin accounts must be pmi-ops accounts using two-factor auth. For prod, only
these accounts are allowed; for development environments, additional accounts
may have access for convenience.

### Config Updates

The `/config` endpoint uses separate auth from any other endpoint. It depends on
the hardcoded values in `rest-api/config/config_admins.json`. If an app ID is
listed in `config_admins.json`, only the service account that it's mapped to
may make `/config` requests. Otherwise, a default
`configurator@$APPID.iam.gserviceaccount.com` has permission.

Config updates happen automatically on deploy for some environments, controlled
by `circle.yml`. To manually update configs, download the appropriate service
account's private key in JSON format (or generate a new key which you can revoke
after use), and pass it to `install_config.sh`.

### Accessing the API

See `rdr_client/client.py` for a Python example of authenticated API access.

To construct authorized request headers on the command line:

```Shell
gcloud auth activate-service-account --key-file=your-key-file.json
gcloud auth print-access-token
--> ya...<rest of token>
```

Then issue a request supplying this token as a bearer token in an authorization header, like

```
GET /rdr/v1/Participant
Authorization: Bearer ya...<rest of token>
```

## Importing codebooks

Codebooks are managed in Google Sheets; the PMI questionnaire codebook is
[here](https://docs.google.com/spreadsheets/d/1b1lmf2KywIVx3-WJBnsR21ImF6Bl345n5VMSdUNBwQI/edit).

Codebooks are published to github as JSON. When you run rdr_client/import_codebook.py, this JSON
is fetched and imported into the SQL database for use in RDR. Existing codes in the codebook are
updated, new codes not in the database are inserted, and codes in the database but not in the
code book are left untouched.

## API Endpoints

All endpoints except `/config` are authenticated using service account
credentials in oauth request headers. The config loaded into an app's datastore
(from `config/config_$ENV.json`) maps service accounts to roles, and the
`auth_*` decorators (from `api_util.py`) assign permissible roles to endpoints.

The config may also specify that a service account is only authorized from
certain IP ranges, or from specific appids (for AppEngine-to-AppEngine
requests), as second verification of the service account's auth.

### Deploying to staging

*   Go to https://github.com/all-of-us/raw-data-repository/releases/new
*   Enter a tag name of the form vX-Y-rcZZ -- e.g. v0-1-rc14. For cherry picks,
    add an additional letter (rc14a).
*   Unless this is intended to be pushed to prod eventually, check the
    "This is a pre-release" box.
*   Submit.

CircleCI should automatically push to staging, based on logic found in
[circle.yml](../circle.yml).

If you are adding new indexes, the tests may fail when they aren't ready yet;
use Rebuild in CircleCI to retry.

CircleCI will also open a new JIRA issue to track the release (or update an
existing issue for cherry picks).

### Tools

Please see the [Tools README](tools/README.md) for more information on command line tools.

# Biobank ID prefixes

Biobank receives samples for participants from multiple RDR environments in its production
environment. In order to ensure there aren't any collisions, we tack on a unique prefix to
client biobank IDs in each environment; the values are:

"A" - production (post-launch)
"B" - dry run / stable
"C" - production (pre-launch)
"X" - staging
"Y" - test
"Z" - dev

Note that these values are only present in our config and responses and Biobank reconciliation
report. They are not actually stored in the database.
