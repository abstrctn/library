NYT Library [![CircleCI](https://circleci.com/gh/newsdev/nyt-library/tree/master.svg?style=svg&circle-***REMOVED***)](https://circleci.com/gh/newsdev/nyt-library/tree/master)
========

A collaborative newsroom documentation site, powered by Google Docs.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Demo Site & User Guide](#demo-site--user-guide)
- [Development Workflow](#development-workflow)
  - [Tests](#tests)
- [Customization](#customization)
- [Deploying the app](#deploying-the-app)
  - [Using Heroku](#using-heroku)
  - [Using Docker](#using-docker)
  - [Standard Deployment](#standard-deployment)
- [App structure](#app-structure)
  - [Server](#server)
  - [Views](#views)
  - [Doc parsing](#doc-parsing)
  - [Listing the drive](#listing-the-drive)
  - [Auth](#auth)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Demo Site & User Guide

A working (read only) demo of Library is available **TKTK**. The site contains
instructions for how to get the most out of Library.

## Development Workflow

1. Clone and `cd` into the repo:

   `git clone git@github.com:newsdev/nyt-library.git && cd nyt-library`


2. From the Google API console, create or select a project, then create a service account with the Cloud Datastore User role. It should have API access to Drive and Cloud Datastore. Store these credentials in `server/.auth.json`.

   - To use oAuth, you will also need to create oAuth credentials.

3. Install dependencies:

   `npm install --no-optional`

4. Create a `.env` file at the project root. An example `.env` might look like

```bash
NODE_ENV=development # node environment
# Google oAuth credentials
GOOGLE_CLIENT_ID=123456-abcdefg.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=abcxyz12345
APPROVED_DOMAINS="nytimes.com,dailypennsylvanian.com" # comma separated list of approved access domains.
SESSION_SECRET=supersecretvalue

# Google drive Configuration
DRIVE_TYPE=team # or folder, if using a folder instead of a team drive
DRIVE_ID=0123456ABCDEF # the ID of your team's drive or shared folder. The string of random numbers and letters at the end of your team drive or folder url.
```
Make sure to remove all comments after the `DRIVE_TYPE` and `DRIVE_ID` vars.

5. Start the app:

   `npm run build && npm run watch`

The app should now be running at `localhost:3000`. Note that Library requires Node v8 or higher.

### Tests
You can run functional and unit tests, which test HTML parsing and routing logic, with `npm test`. A coverage report can be generated by running `npm run test:cover`.

The HTML parsing tests are based on the [Supported Formats doc](https://docs.google.com/document/d/10o-sZt7kzP-GZDEFrNbfwBy7hFe1toNgEVH2QdQSZ5s).  To download a fresh copy of the HTML after making edits, run `node test/data/update.js`.

## Customization
Styles, text, middleware, caching logic, and middleware can be customized to
match the branding of your organization. This is covered in the [customization readme](https://github.com/newsdev/nyt-library/blob/master/custom/README.md).


## Deploying the app

### Using Heroku

Use this button to quickly deploy to Heroku:

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/nytimes/library/)

To set your app's `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLIENT_ID`, and `GOOGLE_CLIENT_SECRET` in Heroku, [set up a Google service account](https://console.cloud.google.com/iam-admin/serviceaccounts) and [OAuth 2.0 client](https://developers.google.com/identity/protocols/OAuth2). Add *<your-heroku-app-url\>.com* as an authorized domain in the general OAuth consent screen setup and then add *http://<your-heroku-app-url\>.com/auth/redirect* as the callback url in the OAuth credential setup itself. Set up your service account with API access to Drive and Cloud Datastore.

If you wish to deploy Library with customizations, create a git repo with the files you would like to include. Set the `CUSTOMIZATION_GIT_REPO` environment variable to the cloning URL. Files in the repo and packages specified in the `package.json` will be included in your library installation.

### Using Docker
Library can be used as a base image for deployment using Docker. This allows you
to automate building and deploying a custom version of Library during Docker's
build phase. If you create a repo with the contents of your `custom` folder, you
could deploy library from that repo with a Dockerfile like the following:

```Dockerfile
FROM TK-DOCKER-BASE-IMAGE

# copy custom files to library's custom repo
COPY . ./custom/

# move to a temporary folder install custom npm packages
WORKDIR /usr/src/tmp
COPY package*.json .npmrc ./
RUN npm i
# copy node modules required by custom node modules
RUN yes | cp -rf ./node_modules/* /usr/src/app/node_modules

# return to app directory and build
WORKDIR /usr/src/app
RUN npm run build

# start app
CMD [ "npm", "start" ]
```

### Standard Deployment
Library is a standard node app, so it can be deployed just about anywhere. If you are looking to deploy to a standard VPS, [Digital Ocean's tutorials](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-16-04) are a great resource.

## App structure

### Server
The main entry point to the app is `index.js`.

This file contains the a express server which will respond to requests for docs
in the configured team drive or shared folder. Additionally, it contains logic
about issuing 404s and selecting the template to use based on the path.

### Views
Views (layouts) are located in the `layouts` folder. They use the `.ejs`
extension, which uses a syntax similar to underscore templates.

Base styles for the views are in the `styles` directory containing Sass files.
These files are compiled to CSS and placed in `public/css`.

### Doc parsing
Doc HTML fetch and parsing is handled by `docs.js`. `fetchDoc` takes the ID of a
Google doc and a callback, then passes the HTML of the document into the
callback once it has ben downloaded and processed.

### Listing the drive
Traversing the contents of the NYT Docs folder is handled by `list.js`. There
are two exported functions:

* `getTree` is an async call that returns a nested hash (tree) of Google Drive
  Folder IDs mapped to their children. It is used by the server to determine
  whether a route is valid or not.

* `getMeta` synchronously returns a hash of Google Doc IDs to metadata objects
  that were saved in the course of populating the tree. This metadata includes
  edit history, document authors, and parent folders.

The tree and file metadata are repopulated into memory on an interval (currently 60s). Calling getTree multiple times will not return fresher data.

### Auth

Authentication with the Google Drive v3 api is handled by the auth.js file, which exposes a single method `getAuth`. `getAuth` will either return an already instantiated authentication client or produce a fresh one. Calling `getAuth` multiple times will not produce a new authentication client if the credentials have expired; we should build this into the auth.js file later to automatically refresh the credentials on some sort of interval to prevent them from expiring.
