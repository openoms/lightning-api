# Lightning Network Daemon API Documentation Site
API Documentation for the Lightning Network Daemon, powered by
[Slate](https://github.com/lord/slate)

## Overview

This repository contains functionality for programmatically pulling API
information from `lncli -h` and `rpc.proto` on the lnd Github, using a Python
script and Jinja2 template to output Slate markdown, which itself generates the
fully rendered static site. 

Pay special attention to these files:
- `templates/index_templates.md`: The Jinja2 template fed into the Python
  script, holding the basic format and introductory information for the site
- `update_and_render.sh`: Update local docs to the latest version available
- `render.py`: The Python script that uses local docs and Jinja template to
  output Slate markdown
- `deploy.sh`: Build static site from Slate markdown and deploy to Google Cloud
  Platform

## Running the site locally

### Prerequisites

You're going to need:

 - **Linux or OS X** — Windows may work, but is unsupported.
 - **Ruby, version 2.2.5 or newer**
 - **Python, version 3 or newer**
 - **Bundler** — If Ruby is already installed, but the `bundle` command doesn't work, just run `gem install bundler` in a terminal.

### Running locally

```shell
git clone https://github.com/lightninglabs/lightning-api

# Start a local server for testing purposes
bundle install --path vendor/bundle
bundle exec middleman server
```

You can now see the docs at `http://localhost:4567`.

## Regenerating documentation

```shell
# Install Jinja for python templating.
pip3 install Jinja2

# Set your $GOPATH to a directory that has access to lncli.
export GOPATH="/path/to/lnd"
export PATH=$PATH:$GOPATH/bin

# Install annotations and protoc-gen-doc.
go get -u github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis
go get -u github.com/pseudomuto/protoc-gen-doc/cmd/protoc-gen-doc

# Get the latest rpc.proto.
curl -o rpc.proto -s https://raw.githubusercontent.com/lightningnetwork/lnd/master/lnrpc/rpc.proto
curl -o rpc.swagger.json -s https://raw.githubusercontent.com/lightningnetwork/lnd/master/lnrpc/rpc.swagger.json

# Generate the rpc.json file from rpc.proto, so that render.py can parse it.
protoc -I. -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis --doc_out=json,rpc.json:. rpc.proto
```

Now, ensure that you have [`lnd` installed](https://dev.lightning.community/guides/installation/)
and your `$GOPATH` set, so that `lncli` is available from the command line.
Let's run the script to render our local Slate docs:
```shell
python3 render.py
```

Now that you're all set up, you can just run `./update_and_render.sh` to
automatically pull the latest rpc.proto and render the local Slate docs.

## Deployment

The Lightning API is deployed with Google Cloud Platform. Visit [this blog
post](https://little418.com/2015/07/jekyll-google-cloud-storage.html) for more
information.

### Steps

1. Install Google Cloud SDK and authenticate into it:
```bash
brew cask install google-cloud-sdk
gcloud auth login
```

2. Build
```bash
bundle exec middleman build --clean
```

3. Push to Google Cloud Bucket
```bash
# -m use faster multithreaded uploads
# -d delete remote files that aren't in the source
# -r recurse into source subdirectories
gsutil -m rsync -d -r ./_site gs://api.lightning.community
```

In the future, you can just run `./deploy.sh` to deploy automatically.

## Automatic Updates and Deployment

Updates to the protobuf definitions can be detected by running the Flask server
within `app.py`.

### Running the server locally

The server uses Flask in order to receive POST requests from GitHub whenever a
new commit has been pushed to the respository. These POST requests will include
the HMAC of a secret token set up within the Webhook settings of a repository.
This token will need to be exported so that the server can verify the request
from GitHub has been authenticated.

First, ensure that `Flask` is installed with:
```shell
$ pip3 install flask
```

Then, the server can be run with:

```shell
$ export WEBHOOK_SECRET_TOKEN=YOUR_TOKEN_HERE
$ FLASK_APP=app.py
$ flask run
```

Once a POST request from GitHub has been received, the server will check if
there were any commits which included a change to the protobuf definitions. If
there was, then the documentation will be automatically regenerated and
deployed.
