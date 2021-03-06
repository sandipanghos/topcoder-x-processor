## Requirements

- Nodejs 8 is required
- [Apache Kafka](https://kafka.apache.org/)
- MongoDB 3.4

## Install dependencies

```shell
npm install
```

## Source code lint

eslint is used to lint the javascript source code:

```shell
npm run lint
```

## Configuration

The following config parameters are supported, they are defined in `config/default.js` and can be configured in system environment:


| Name                           | Description                                | Default                          |
| :----------------------------- | :----------------------------------------: | :------------------------------: |
| LOG_LEVEL                      | the log level                              |  debug                           |
| TOPIC                          | the kafka subscribe topic name             |  tc-x-events                    |
| PARTITION                  | the kafka partition            |  0|
| KAFKA_OPTIONS                  | the connection option for kafka            |  see below about KAFKA options                  |
| MONGODB_URL  | the MongoDB URL which must be same as Topcoder x tool | mongodb://127.0.0.1:27017/topcoderx |
|TC_DEV_ENV| the flag whether to use topcoder development api or production| false|
| TC_AUTHN_URL | the Topcoder authentication url | https://topcoder-dev.auth0.com/oauth/ro |
| TC_AUTHN_REQUEST_BODY | the Topcoder authentication request body. This makes use of some environment variables: `TC_USERNAME`, `TC_PASSWORD`, `TC_CLIENT_ID`, `CLIENT_V2CONNECTION` | see `default.js` |
| TC_AUTHZ_URL | the Topcoder authorization url | https://api.topcoder-dev.com/v3/authorizations |
| NEW_CHALLENGE_TEMPLATE | the body template for new challenge request. You can change the subTrack, reviewTypes, technologies, .. here | see `default.js` |
| NEW_CHALLENGE_DURATION_IN_DAYS | the duration of new challenge | 5 |
| NODE_MAILER_OPTIONS| the node mailer smtp options, see [here](https://nodemailer.com/smtp/ for more detail)| see `default.js` |
|EMAIL_SENDER_ADDRESS| the email sender email address||
|ISSUE_BID_EMAIL_RECEIVER| the email receiver about bid email||
|TC_URL| the base URL of topcoder to get the challenge URL| defaults to `https://www.topcoder-dev.com`|
|GITLAB_API_BASE_URL| the URL for gitlab host| defaults to `https://gitlab.com`|

KAFKA_OPTIONS should be object as described in https://github.com/SOHU-Co/kafka-node#kafkaclient
For using with SSL, the options should be as
```
 {
    kafkaHost: '<server>',
    sslOptions: {
      cert: '<certificate>', 
      key:  '<key>'
    }
 }
```

## Local Deployment

```shell
npm start
```

## Setup for verification
Before verifying the tool, 3 service needs be configured and run them
- processor
- receiver
- Topcoder X

Go to Topcoder X UI login with above used topcoder username and
- go to settings and make sure git hosts are correctly setup, if not click setup and authorize to setup.

- Go to Topcoder X UI and go to project management and add a project from git account and click save, and edit the same project and click 'Add Webhooks' button, verify that webhooks are set up correctly on git host's project.

Now, receiver service can receive the webhooks from git host's project and processor can processes the requests. Now you can verify this service by following the verification steps below

## Verification

- create an issue in the repo, you can see the logs in `receiver` and `processor`, the `issue.created` event is generated.
- update an issue in the repo, you can see the logs in `receiver` and `processor`, the `issue.updated` event is generated.
- create a comment on an issue, you can see the logs in `receiver` and `processor`, the `comment.created` event is generated.
- update a comment on an issue, you can see the logs in `receiver` and `processor`, the `comment.updated` event is generated.
- assigned a user to an issue, you can see the logs in `receiver` and `processor`, the `issue.assigned` event is generated.
- unassigned a user to an issue, you can see the logs in `receiver` and `processor`, the `issue.unassigned` event is generated.
- add or remove a label to an issue, you can see the logs in `receiver` and `processor`, the `issue.labelUpdated` event is generated.
- create a pull request, you can see the logs in `receiver` and `processor`, the `pull_request.created` event is generated.
- close a pull request without merge, you can see the logs in `receiver` and `processor`, the `pull_request.closed` event is generated and the `merged` property is `false`.
- merge a pull request, you can see the logs in `receiver` and `processor`, the `pull_request.closed` event is generated and the `merged` property is `true`.

### Create a new challenge for a new issue
- Create a new issue in the repo. E.g.
  - With the title: [$20] A new issue title
  - With the body (Markdown supported):
    ```
    ## A new issue body header

    A new issue body content with ***bold*** text and a [link](https://github.com/ngoctay/Luettich-test/issues/).
    ```
- You will see a new project created in the logs if project for that repository is not created already. E.g.
  ```
  debug: new project created with id 11916 for issue 3
  ```
- Then a new challenge created in the logs. E.g.
  ```
  debug: new challenge created with id 30051089 for issue 3
  ```
- Wait a minute or 2 for Topcoder internal systems to process the new challenge. You may get an error page `HTTP Status 404` if the internal processing haven't completed yet.
- Visit challenge url to verify the challenge:
  ```
  https://www.topcoder-dev.com/challenge-details/30051089/?type=develop&noncache=true
  ```
  ***NOTE***: change 30051089 to the your challenge id created in the previous step

### Update the challenge when the issue was updated

- Update title of the issue above. E.g.
  - With the title: [$50] A new issue title - Updated the prize
- Wait a minute or 2
- Visit the challenge url to verify the updated prize and title

- Update the body of the issue. E.g.
  ```
  ## A new issue body header - Updated

  A new issue body content with ***bold*** text and a [link](https://github.com/ngoctay/Luettich-test/issues/). Updated
  ```
- Wait a minute or 2
- Visit the challenge url to verify the updated body

- Now try to update the title by adding a space after $50: [$50 ] A new issue title - Updated the prize
- The event will be ignored by processor:
  ```
  debug: nothing changed for issue 3
  ```

- Update the tilte by removing `[$50 ]`, you'll get an error:
  ```
  error:  Error: Cannot parse prize from title: A new issue title - Updated the prize
  ```

### Process Bidding Comments
- Add/update comment to bid or accept the bid
- To bid on the issue, comment as `/bid $50`. Minimum one space is need between /bid and $ sign. This will trigger `comment.created` or `comment.updated` event based on comment create/edit.
- if comment is bid then an email is sent to the user configured in `ISSUE_BID_EMAIL_RECEIVER` in config file
- To accept bid on the issue, comment as `/accept_bid @username $50`
- if comment is accepting bid then 
  - title of issue will be changed with accepted bid amount
  - issue will be assigned to user `username` if not already assigned

### Issue Assignment
When an user is assigned to an issue then 'issue.assigned' event will be captured and processed as
- processor will get the Topcoder handle for such user using Ragnar self service tool api
- if user is found then the topcoder challenge associated with that issue will be updated as
  - topcoder user will be assigned
- if user is not found in mapping
  - comment on github/gitlab issue will added as :
  ```
  @username, please sign-up with Topcoder x Self-service tool.
  ```
  - user will be unassigned from issue
