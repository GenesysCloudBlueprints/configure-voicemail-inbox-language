---
title: Configuring a user’s VM Inbox Language
author: barrez.bert
indextype: blueprint
icon: blueprint
image: images/CallBlacklist.gif
category: 4
summary: |
  This blueprint addresses the need to configure voicemail inbox language per user profile, rather than system-wide.
---
This blueprint is a solution for idea AMUGM-I-76 (https://genesyscloud.ideas.aha.io/ideas/AMUGM-I-76). This blueprint addresses the need to configure voicemail inbox language per user profile, rather than system-wide. This is particularly useful for environments with multilingual users, such as English and French speakers.

We can configure per user based on their division or if they have a primary number, we can do it by the international dialling code. This will only work if TTS is available for that language.

## Solution components

* **Genesys Cloud** - A suite of Genesys cloud services for enterprise-grade communications, collaboration, and contact center management. Contact center agents use the Genesys Cloud user interface.
* **Genesys Cloud API** - A set of RESTful APIs that enables you to extend and customize your Genesys Cloud environment.
* **Data Action** - Provides the integration point to invoke a third-party REST web service or AWS lambda.
* **Architect flows** - A flow in Architect, a drag and drop web-based design tool, dictates how Genesys Cloud handles inbound or outbound interactions.
* **Sites** - Configure your site to push all the users that dial *86 into your inbound call.

## Prerequisites

### Specialized knowledge

* Administrator-level knowledge of Genesys Cloud
* Expereince with REST API authentication

### Genesys Cloud account

* A Genesys Cloud CX 1 license. For more information, see [Genesys Cloud Pricing](https://www.genesys.com/pricing "Opens the Genesys Cloud pricing article").
* The Master Admin role in Genesys Cloud. For more information, see [Roles and permissions overview](https://help.mypurecloud.com/?p=24360 "Opens the Roles and permissions overview article") in the Genesys Cloud Resource Center.

## Step 1: Create a Data Action

* Create a Genesys Cloud Data Action: This action will retrieve the user's division based on their email. Ensure you have an active Genesys Cloud data actions integration with the appropriate permissions.
* Import the [data action](../Get-Users-Division-From-Email.json)
* Save and Publish the data action

## Step 2: Inbound Call Flow
* In this case I have named my inbound call flow SetVoicemailLanguage.
* You can import the [Inbound Call Flow](../SetVoicemailLanguage_v1-0.i3InboundFlow).

   ![create a data table](images/datatable.gif "create a data table")

* Create your own Inbound Call Flow by following the steps below.
* In your inbound call flow create a reusable task and give it a name.
* Add an Update Data Block, you will need to add 3 string statements.

| Variable Name | Value to Assign |
|---------------|-----------------|
| Flow.language | e.g. en-gb (default language) |
| Flow.UserSIP  | If(ToPhoneNumber(Call.Ani).isSip,Call.Ani,"") |
| Flow.CallerNumber | If(ToPhoneNumber(Call.Ani).isTel,ToPhoneNumber(Call.Ani).e164,"") | 

* The first is **Flow.language** this is for setting a default language to use if there is no specific language set for that division/international dialing code.
* The second **Flow.UserSIP** is to check if the Call.Ani is a sip address, we will manipulate this later to return the users email.
* The third **Flow.CallerNumber** is to check whether the Call.Ani is a telephone number and to return the number in e164 format if it is.

* Picture 1

* Picture 2

* Add a **Decision Block** with the following: 

| Decision | 
|---------------|
| IsNotSetOrEmpty(Flow.CallerNumber) | 

* Picutre 3

* Nothing is required under the No path

* Picture 4

* Under the Yes path add an **Update Data Block** and **add 3 string statements**.
* Here we manipulate the **Flow.UserSIP** to convert it to the users email address.

| Variable Name | Value to Assign |
|---------------|-----------------|
| Flow.UserEmail | Left(Flow.UserSIP,Length(Flow.UserSIP)-10) |
| Flow.UserEmail | Right(Flow.UserEmail,Length(Flow.UserEmail)-4) |
| Flow.UserEmail | Replace(Flow.UserEmail,"%40","@") | 

* Picture 5

* Picture 6

* Still under the Yes path, add a **Call Data Action**.
* Select the **category** that the “Get Users Division From Email” is under in this case it is Genesys Cloud Data Actions. 
* Data Action - **“Get Users Division From Email"**.

| Inputs for the Data Action | Value to Assign |
|----------------------------|-----------------|
| email | Flow.UserEmail |

| Success Outputs | Value to Assign |
|----------------------------|-----------------|
| results__division__name | Flow.Division | 

* Picture 7

* Picture 8

* **Outside of the decision block**, you can add a **switch block** for determining which language is to be used in the users voicemail inbox either based on their international dialling code or the users division.

* Picture 9

* In the cases you can include the following:

| Case | Value to Assign |
|------|-----------------|
| 1 | (IsNotSetOrEmpty(Flow.Division),false,Contains(Flow.Division,"Input division name here")) or If(IsNotSetOrEmpty(Flow.CallerNumber),false,ToPhoneNumber(Call.Ani).dialingCode == "input international dialling code here") | 
| 2 | If(IsNotSetOrEmpty(Flow.Division),false,Contains(Flow.Division,"Input division name here")) or If(IsNotSetOrEmpty(Flow.CallerNumber),false,ToPhoneNumber(Call.Ani).dialingCode == "input international dialling code here") |
| 3 | If(IsNotSetOrEmpty(Flow.Division),false,Contains(Flow.Division,"Input division name here")) or If(IsNotSetOrEmpty(Flow.CallerNumber),false,ToPhoneNumber(Call.Ani).dialingCode == "input international dialling code here") |

* Picture 10

* Under each case you will add an **Update Data Block**, you will **add a string statement** to update the **Flow.language** with the appropriate language required

| Variable Name | Value to Assign |
|---------------|-----------------|
| Flow.language | e.g. en-gb |

* You can repeat this until you have included all required languages. You can leave the default path blank as you have set the default language at the start of the flow.

* Picture 11

* **Outside of the switch**, you will add a **Transfer to Number Block**.

* Picture 12

* Picture 13

* In the number input, input the following: "sip:*86@127.0.0.1;language="+ Flow.language + ";user=voicemailbox"

* Add a **Disconnect Block** at the end of the inbound call flow.

* Picture 14

* Publish the flow.

## Step 3: Configuring the site

* Here you will configure your site to push all users that dial *86 into your inbound call flow that you created.

1. In your **Site**, click on **Number Plans**.
2. Click on **+ New Number Plan**
3. **Number Plan name**: VM.
4. **Match Type**: Regular Expression. 
5. **Match Expression**: ^\*86$
6. **Normalized Number Expression** will be the name of your inbound call flow, I have named my inbound call flow SetVoicemailLanguage: sip:SetVoicemailLanguage@localhost
7. **Classification**: VM.
8. Click **Save Number Plans**

* Picture 15

* Make a test call from your phone to *86 to make sure the language that you configured is being used.

## Additional resources

* [Genesys Cloud API Explorer](https://developer.genesys.cloud/devapps/api-explorer "Opens the GC API Explorer") in the Genesys Cloud Developer Center
* [Genesys Cloud notification triggers](https://developer.genesys.cloud/notificationsalerts/notifications/available-topics "Opens the Available topics page") in the Genesys Cloud Developer Center
* The [ani-blacklist](https://github.com/GenesysCloudBlueprints/ani-blacklist) repository in GitHub
