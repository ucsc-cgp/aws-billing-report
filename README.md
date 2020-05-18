# AWS Billing Report

Given AWS billing data stored in an S3 bucket and a MySQL database, compiles
a report with historical expense information for a number of specified AWS
accounts and sends that report to a number of recipients.

You can install dependencies with

    $ cpanm --installdeps .

and run the script with

    $ perl report_aws_spending


## Configuration

Specify the people who should receive the report in `recipients`. You can refer
to `recipients.example` to see how the file should look like - each recipient
should be separated in the file by a newline.

Specify the accounts to check in `accounts.csv`. Similar to `recipients`, you
can reference `accounts.csv.example` to see what the file should look like.
Each account should be specified on its own line, starting with the account
name then the account number, each separated by a comma.

Both the `recipients` and `accounts.csv` file should be placed in
`/root/aws-reporting/`.

The following need to be configured directly in the script:
* AWS access keys (`$access_key`, `$secret_key`)
* Billing data S3 bucket name (`$bucket`)
* MySQL account credentials and connection info (`$host`, `$pw`, `$mysqluser`,
  `$database`)
* The `from` email address (`$from`)


## Development

To run this script locally, you'll need a few things:
* A `mysqldump` of the database that stores historical spend information (that
  this script accesses). If you don't have this, it may be sufficient to create
  a table for each account in `accounts.csv`. Store this file in a subdirectory
  named `db/`; `docker-compose` will automatically populate the database with
  this data.
* AWS keys with access to billing data in S3
* `docker-compose`

MySQL connection details need to be configured in `report_aws_spending`
manually:
* Change the hostname to `db`
* Change the username to `root`
* Change the password to `hunter2`, or whatever is specified in
  `docker-compose.yml`

Then:

    $ docker-compose up --abort-on-container-exit

will run the script. It will take about five minutes to run once the database
service has spun up entirely. Maybe a little longer than that.

Before running the script, the script should be modified to output the report
to a file, or something else besides piping it to `sendmail`. The Docker
container does not have postfix installed, so without this change, the script
will fail. This can be accomplished by replacing

    open(MAILSEND, "|/usr/sbin/sendmail -t");

with something like

    open(MAILSEND, "| tee report.html");

You can then open the generated report in your browser.


### Known bugs

If the report service starts before the database service is finished starting,
the script will fail and restart until it succeeds. Each restart downloads
files from S3, so be cognizant of this and kill it if it isn't working.

This should only be a problem the first time the script is run, or if anonymous
volumes are manually purged (as with `docker up --renew-anon-volumes`).

### Without docker-compose

If you're installing without using Docker, you can install dependencies with

    $ cpanm --installdeps .

It will be useful to look at the install process outlined in the `Dockerfile`.

Be careful replicating the development environment on your local machine,
especially if you are running macOS. One of the dependencies
(LWP::Protocol::https) appears to, in some cases, interfere with an existing
installation of openssl. (This can break tools that communicate over https.)

If you're committed to the endeavor, MySQL bindings need to be installed and
need special configuration to work. On macOS, you need to `brew install
mysql-connector-c` and `brew install mysql` and link/unlink both of them in
some arcane order so you can configure MySQL in the right way so that cpan
knows to look for openssl libraries in the right place. (This exercise is left
to the reader.)

### Testing

You might encounter issues testing this script locally depending on what
recipients are specified. (For example, Google will not accept mail from some
hosts over IPv6 without extra configuration.) In this case, the script can be
modified to output the email report to a file.
