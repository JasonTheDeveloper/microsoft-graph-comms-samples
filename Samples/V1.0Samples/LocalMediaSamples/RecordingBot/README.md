# Introduction

## Note

The system will load the bot and join it to appropriate calls and meetings in order for the bot to enforce compliance with the administrative set policy.

## About

The Policy Recording bot sample guides you through building, deploying and testing a bot. This sample demonstrates how a bot can receive media streams for recording. Please note that the sample does not actually record. This logic is left up to the developer.

## Getting Started

This section walks you through the process of deploying and testing the sample bot.

### Bot Registration

1. Follow the steps in [Register Calling Bot](https://microsoftgraph.github.io/microsoft-graph-comms-samples/docs/articles/calls/register-calling-bot.html). Save the bot name, bot app id and bot secret for configuration.
    * For the calling webhook, by default the notification will go to https://{your domain}/api/calling. This is configured with the `CallSignalingRoutePrefix` in [HttpRouteConstants.cs](https://github.com/microsoftgraph/microsoft-graph-comms-samples/blob/master/Samples/BetaSamples/LocalMediaSamples/PolicyRecordingBot/FrontEnd/Http/Controllers/HttpRouteConstants.cs).
    * Ignore the "Register bot in Microsoft Teams" section as the Policy Recording bot won't be called directly. These bots are related to the policies discussed below, and are "attached" to users, and will be automatically invited to the call.

1. Add the following Application Permissions to the bot:

    * Calls.AccessMedia.All
    * Calls.JoinGroupCall.All
   
1. The permission needs to be consented by tenant admin. Go to "https://login.microsoftonline.com/common/adminconsent?client_id=<app_id>&state=<any_number>&redirect_uri=<any_callback_url>" using tenant admin to sign-in, then consent for the whole tenant.

### Create an Application Instance

Open powershell (in admin mode) and run the following commands. When prompted for authentication, login with the tenant admin.
  * `> Import-Module SkypeOnlineConnector`
  * `> $sfbSession = New-CsOnlineSession -Verbose`
  * `> Import-PSSession $sfbSession`
  * `> New-CsOnlineApplicationInstance -UserPrincipalName <upn@contoso.com> -DisplayName <displayName> -ApplicationId <your_botappId>`
  * `> Sync-CsOnlineApplicationInstance -ObjectId <objectId>`

### Create a Recording Policy
Requires the application instance ID created above. Continue your powershell session and run the following commands.
  * `> New-CsTeamsComplianceRecordingPolicy -Enabled $true -Description "Test policy created by <yourName>" <policyIdentity>`
  * ```> Set-CsTeamsComplianceRecordingPolicy -Identity <policyIdentity> -ComplianceRecordingApplications ` @(New-CsTeamsComplianceRecordingApplication -Parent <policyIdentity> -Id <objectId>)```

After 30-60 seconds, the policy should show up. To verify your policy was created correctly:
  * `> Get-CsTeamsComplianceRecordingPolicy <policyIdentity>`

### Assign the Recording Policy
Requries the policy identity created above. Contine your powershell session and run the following commands.
  * `> Grant-CsTeamsComplianceRecordingPolicy -Identity <userUnderPolicy@contoso.com> -PolicyName <policyIdentity>`

To verify your policy was assigned correctly:
  * `> Get-CsOnlineUser <userUnderPolicy@contoso.com> | ft sipaddress, tenantid, TeamsComplianceRecordingPolicy`

### Prerequisites

* Install the prerequisites:
    * [Visual Studio 2017+](https://visualstudio.microsoft.com/downloads/)
    * [PostMan](https://chrome.google.com/webstore/detail/postman/fhbjgbiflinjbdggehcddcbncdddomop)
    * [Ngrok](https://ngrok.com/download)
      * **Note:** You will need a paid pro account.

### Test Locally

#### Nrgok

1. Navigate to [Reserved Domains](https://dashboard.ngrok.com/endpoints/domains) in your Ngrok account and reserve a domain. We will configure Azure and the bot to point to this domain.

2. Now navigate to [TCP Addresses](https://dashboard.ngrok.com/endpoints/tcp-addresses) and reserve a TCP port. This will be used to push incoming streams to.

3. With those two reserved, now it's time to configure ngrok on our local machine to forward our newly configured endpoints to localhost running on our machines.

    Create a new `ngrok.yaml` then paste the following in that file:

    ```yaml
      authtoken: YOUR_AUTH_TOKEN
      tunnels:
      signaling:
        addr: "https://localhost:9441"
        proto: http
        subdomain: YOUR_NGROK_SUBDOMAIN
        host_header: "localhost:9441"
      media: 
        addr: 8445
        proto: tcp
        remote_addr: "YOUR_RESERVED_TCP_ADDRESS_INCLUDING_PORT"

    ```

    Make sure you replace:

    * `YOUR_AUTH_TOKEN` with your Ngrok auth token.
    * `YOUR_NGROK_SUBDOMAIN` with the subdomain you reserved in Ngrok in step 1.
    * `YOUR_RESERVED_TCP_ADDRESS_INCLUDING_PORT` with the full TCP reserved address (including the port) created in step 2. For example, `1.tcp.ngrok.io:1111`.

4. Now with that configured, it's time to run Ngrok. Open up a new terminal instance and run the following command:

    ```cmd
      ngrok start -all -config ngrok.yaml
    ```

    If everything works you should be able to hit the reserved domain from step 1 and see traffic coming through on `localhost:9441`.

5. Generate certificates for your reserved domain and install on your machine. Make sure you copy your certificate's thumbprint - we will need this later. To help with this, see https://github.com/jakkaj/sslngrokdevcontiner.

    - Once you run `sslngrokdevcontiner` you should have certificate `pem` files in `C:\...\sslngrokdevcontiner\letsencrypt\archive\YOUR_NGROK_SUBDOMAIN`
    - To generate a `pfx` file run

      ```bash
        openssl pkcs12 -inkey privkey1.pem -in cert1.pem -export -out cert.pfx
      ```
      - To access `openssl` you may need to install [OpenSSL](https://chocolatey.org/packages/openssl) or use WSL/Ubuntu/bash
    - To import the certificate, double click on `cert.pfx`
    - The certificate thumbprint can be found by running "Manager user certificates" or "Manager computer certificates", locating your certificate, opening it, clicking on the "Details" tab, and scrolling to "Thumbprint"

6. Open a SSL port with the command in `cmd`:

    ```cmd
      netsh http add sslcert ipport=0.0.0.0:9441 certhash=YOUR_CERTIFICATE_THUMBPRINT appid={YOUR_APP_ID}
    ```

    As an example:

    ```cmd
      netsh http add sslcert ipport=0.0.0.0:9441 certhash=7339d0267af852084a6db7d0f70b6034cfc576fc appid={7775d4a2-c831-4602-80fb-608f6a1c1b8c}
    ```

#### RecordingBot

1. Under `Entry`, create a new file called `App.Secrets.config` and copy the following:

    ```xml
      <appSettings>
        <add key="BotName" value="%BotName%" />
        <add key="AadAppId" value="%AppId%" />
        <add key="AadAppSecret" value="%AppSecret%" />
        <add key="ServiceDnsName" value="%ServiceDns%" />
        <add key="CertificateThumbprint" value="ABC0000000000000000000000000000000000CBA" />
        <add key="InstancePublicPort" value="%PublicTCPPort%" />
      </appSettings>
    ```

    Make sure you replace the following:

    * `%BotName%` with the name of your Bot Channel Registry name.
    * `%AppId%` your bot ID.
    * `%AppSecret%` your bot's secret.
    * `%ServiceDns%` your reserved domain (e.g. `mybot.ngrok.io`) created in step 1 of section [Ngrok](#nrgok).
    * `ABC0000000000000000000000000000000000CBA` your certificate's thumbprint from step 5 of section [Ngrok](#nrgok).
    * `%PublicTCPPort%` your TCP port (e.g. `11111`) reserved in step 2 of section [Ngrok](#nrgok).

2. Launch `RecordingBot.sln` in Visual Studio, make sure `Entry` is set as default and you can build the project without errors.

3. Make sure Ngrok is running and then run `Entry` from Visual Studio. A console window will popup and you should see a message saying `Hit`. Once the bot is up and running, you'll see the message `Running`. Once you see that, you can test the bot using Teams.

#### Teams

1. Set up the test meeting and test clients:
   1. Sign in to Teams client with a non-recorded test tenant user.
   1. Use another Teams client to sign in with the recorded user. (You could use an private browser window at https://teams.microsoft.com)

1. Place a call from the Teams client with the non-recorded user to the recorded user.

1. Your bot should now receive an incoming call, and join the call (See next step for retrieving logs). Use the recorded user's Teams client to accept the call.

1. Interact with your service, _adjusting the service URL appropriately_.
    1. Get diagnostics data from the bot. Open the links in a browser for auto-refresh. Replace the call id 311a0a00-53d9-4a42-aa78-c10a9ae95213 below with your call id from the first response.
       * Active calls: https://bot.contoso.com/calls
       * Service logs: https://bot.contoso.com/logs

    1. By default, the call will be terminated when the recording status has failed. You can terminate the call through `DELETE`, as needed for testing. Replace the call id `311a0a00-53d9-4a42-aa78-c10a9ae95213` below with your call id from the first response.

        ##### Request
        ```json
            DELETE https://bot.contoso.com/calls/311a0a00-53d9-4a42-aa78-c10a9ae95213
        ```

## Troubleshooting

### ConfigurationErrorsException

If you see:

> System.Configuration.ConfigurationErrorsException: 'The root element must match the name ...

or

> System.Configuration.ConfigurationErrorsException: 'Unrecognized attribute 'file' ...

Please make sure that your `App.Secrets.config` contents matches the sample code in *RecordingBot point 1* and **is not** based on a copy of `App.config`.