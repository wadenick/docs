---
description: Configuring a Custom SMS Provider for MFA using Twilio
topics:
  - mfa
  - sms
  - custom-sms-provider 
contentType:
  - how-to
useCase:
  - customize-mfa
---
# Configuring a Custom SMS Provider for MFA using Twilio

Twilio provides two APIs for sending SMS messages:

  - [Programmable SMS](https://www.twilio.com/sms) is a flexible API designed to fully automate SMS communications.
  - [Verify](https://www.twilio.com/verify) is an API designed to send one-time codes while hides the complexity of SMS delivery. 

Auth0 has built-in support for sending messages through Twilio using the Twilio Programmable SMS API. If you want to write your own custom logic for using that API, or you want to use Twilio Verify, you can configure SMS to use a Phone Message Hook.

## Prerequisites

Before you begin this tutorial, please:

* Sign up for a [Twilio](https://www.twilio.com/try-twilio) account.
* Create a new Messaging Service in the [Programmable SMS console](https://www.twilio.com/console/sms/services) or in the [Verify console](https://www.twilio.com/console/verify/services) depending on the API you want to use.
* Add a phone number that is enabled for SMS to your service and capture the number.
* Capture the Account SID and Authorization Token by clicking *Show API Credentials* in the [Twilio SMS Dashboard](https://www.twilio.com/console/sms/dashboard)

## 1. Create a Send Phone Message hook 

You will need to create a [Send Phone Message](/hooks/extensibility-points/send-phone-message) hook, which will hold the code and secrets of your custom implementation.

::: note
Please note that you can only have ONE Send Phone Message Hook active at a time.
:::

## 2. Configure Hook secrets

Add three [Hook Secrets](/hooks/secrets/create) with key = `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` and `TWILIO_PHONE_NUMBER` with the values previously captured.

## 3. Implement the Hook

If you want to use the Programmable SMS API, [edit](/hooks/update) the Send Phone Message hook code to match the example below.

```js
/**
@param {string} recipient - phone number
@param {string} text - message body
@param {object} context - additional authorization context
@param {string} context.factor_type - 'first' or 'second'
@param {string} context.message_type - 'sms' or 'voice'
@param {string} context.action - 'enrollment' or 'authentication'
@param {string} context.language - language used by login flow
@param {string} context.code - one time password
@param {string} context.ip - ip address
@param {string} context.user_agent - user agent making the authentication request
@param {string} context.client_id - to send different messages depending on the client id
@param {string} context.name - to include it in the SMS message
@param {object} context.client_metadata - metadata from client
@param {object} context.user - To customize messages for the user
@param {function} cb - function (error, response)
*/
module.exports = function(recipient, text, context, cb) {

  const accountSid = context.webtask.secrets.TWILIO_ACCOUNT_SID; 
  const authToken = context.webtask.secrets.TWILIO_AUTH_TOKEN; 
  const fromPhoneNumber = context.webtask.secrets.TWILIO_PHONE_NUMBER;

  const client = require('twilio')(accountSid, authToken); 
 
  client.messages 
      .create({ 
         body: text, 
         from: fromPhoneNumber,       
         to: recipient 
      }) 
      .then(function() {
        cb(null, {});
      }) 
      .catch(function(err) {
        cb(err);
      });
};

```

If you want to use the Verify API, you need to make sure that the Twilio Verify Service is configured to accept a custom code. At the time of writing, you need to contact Twilio support to get it enabled. 

[Edit](/hooks/update) the Send Phone Message hook code to match the example below.

```js
module.exports = function(recipient, text, context, cb) {

  const accountSid = context.webtask.secrets.TWILIO_ACCOUNT_SID; 
  const authToken = context.webtask.secrets.TWILIO_AUTH_TOKEN; 
  const fromPhoneNumber = context.webtask.secrets.TWILIO_PHONE_NUMBER;

  const client = require('twilio')(accountSid, authToken); 
 
  client.verify.services(accountSid)
      .verifications
      .create({
        to: recipient,
        channel: 'sms',
        customCode: context.code
      })
      .then(function() {
        cb(null, {});
      }) 
      .catch(function(err) {
        cb(err);
      });
};
```

## 4. Add the Twilio Node JS Helper NPM package

The hook uses the [Twilio Node.JS Helper Library](https://github.com/twilio/twilio-node). You will need to add the 'twilio-node' module from the **NPM modules** section in the Hooks configuration. You can access it by clicking the icon on the top left of the Hook editor.

## 5. Test your hook implementation

Click the **Run** icon on the top right to test the hook. Edit the parameters to specify the phone number to receive the SMS and click the **Run** button.

## 6. Test the MFA flow

Trigger an MFA flow and double check that everything works as intended. If you do not receive the SMS, please take a look at the [Hook Logs](/hooks/view-logs).

## Troubleshooting

If you do not receive the SMS, please look at the logs for clues and make sure that:

- The Hook is active and the SMS configuration is set to use 'Custom'.
- You have configured the Hook Secrets as per Step 2
- Those are the same ones you created in the Twilio Console
- Your are sending the messages from a phone number that is linked to your Twilio account
- Your phone number is formatted using the [E.164 format](https://en.wikipedia.org/wiki/E.164)
