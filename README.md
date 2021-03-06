iCalForce
=========

iCalendar (.ics) exporter for Salesforce/Force.com.  

You can watch Salesforce's "Event" via
Google calendar, Outlook.com, etc... by subscribing calendar URL.  
You can also import calendar to Microsoft Outlook.

**supported client:**
  * Google calendar
  * Outlook.com
  * Microsoft Outlook
  * other ics readable calendar apps (e.g. iCal, Thunderbird+Lightning, ...)

**supported Salesforce/Force.com editions: EE, UE, DE**

**key features**
  * Export as iCalendar (.ics) format
  * Whitelist access restriction
  * View your calendar URL via Force.com Canvas App
  * View your calendar URL via OAuth SSO App

----
### contents
  * [Warning](#warning)
  * [Setup](#setup)
  * [Defining Connected App on your Salesforce organization](#defining-connected-app-on-your-salesforce-organization)
  * [To Increase Security](#to-increase-security)
    * [Remove /private/secret.php from deploying image](#remove-privatesecretphp-from-deploying-image)
    * [Don't set 'OWNERID'](#dont-set-ownerid)
    * [White-List](#white-list)
    * [Salesforce sharing configurations](#salesforce-sharing-configurations)

----
# Warning
  * This app don't have any user auth mechanism.  
    To keep calendar secret, you should the calendar url secret.  
    And if it is suspected that url is leaked, it is necessary to change the url immediately.

# Setup
*if you want deploying to Heroku, see also [deploy-heroku.md](https://github.com/shellyln/iCalForce/blob/master/deploy-heroku.md).*

  1. Download iCalForce from https://github.com/shellyln/iCalForce.git .
     ```bash
     $ git clone https://github.com/shellyln/iCalForce.git iCalForce.repo
     ```
     or download Zip file from [here](https://github.com/shellyln/iCalForce/archive/master.zip).

  1. Run setup script.
     ```bash
     $ cd iCalForce.repo/iCalForce
     $ bash ./setup.sh
     ```
  1. Set server variables
    * **USERNAME** - Salesforce account's login name.
    * **PASSWORD** - Salesforce account's concatenated login pasword + security token.  
      if password is XXXX and security token is YYYY, you should set PASSWORD XXXXYYYY.
    * **OWNERID** - 15 chars ID of User. it is used when url parameter is omitted.
    * **BASEURL** - base url of your org. default value (if you don't set) is  
      https://ap1.salesforce.com

    **httpd.conf (apache)**
    ```apache
    SetEnv USERNAME alice@example.com
    SetEnv PASSWORD passSecuritytoken
    SetEnv OWNERID  002i1234567Zz7P
    ```
    **nginx.conf (nginx)**
    ```nginx
    server {
        ...
        location ~ \.php(/|$) {
            ...
            fastcgi_param USERNAME alice@example.com;
            fastcgi_param PASSWORD passSecuritytoken;
            fastcgi_param OWNERID  002i1234567Zz7P;
        }
        ...
    }
    ```
  1. [OPTIONAL] Set server variables (for OAuth SSO App / Force.com Canvas App)
    * **CLIENT_ID** - OAuth Client App ID provided from Salesforce
    * **CLIENT_SECRET** - OAuth Client App Secret provided from Salesforce
    
    ID and Secret are get from salesforce [Setup]>[Build]>[Create]>[Apps]>[Connected Apps]>[new].

    **nginx.conf (nginx)**
    ```nginx
    server {
        ...
        location ~ \.php(/|$) {
            ...
            fastcgi_param CLIENT_ID "Akeds9IKLSKLA33KLL432AiokIO93kddskdfcks4_uKIek32_e1.JK";
            fastcgi_param CLIENT_SECRET "123456789012345";
        }
        ...
    }
    ```

  1. Set website's root **iCalForce.repo/iCalForce/public_html**.

    **httpd.conf (apache)**
    ```apache
    ...
    DocumentRoot "/path/to/iCalForce.repo/iCalForce/public_html"
    <Directory "/path/to/iCalForce.repo/iCalForce/public_html">
        ...
    </Directory>
    ...
    ```
    **nginx.conf (nginx)**
    ```nginx
    server {
        ...
        root /path/to/iCalForce.repo/iCalForce/public_html;
        ...
    }
    ```

  1. reboot (or reload config) web server.

  1. access the url

    ```
    https://theapp.your.domain.com/private/secret.php?u=15charsCaseSensitiveUserId
    ```

# Defining **Connected App** on your Salesforce organization
*It is optional feature.*
  1. [Setup]>[Build]>[Create]>[Apps]>[Connected Apps]>[New] and edit as follows:
    * API (Enable OAuth Settings)
      * Enable OAuth Settings: [x]
      * Callback URL: https://theapp.your.domain.com/oa-callback.php
      * Use digital signatures: [ ]
      * Selected OAuth Scopes: id, api, reflesh_token
    * Web App Settings
      * Start URL: https://theapp.your.domain.com/my.php
      * Enable SAML: [ ]
    * Mobile App Settings
      * Start URL: https://theapp.your.domain.com/my.php
      * Pin Protect: [ ]
    * Canvas App Settings
      * Force.com canvas: [x]
      * Canvas App URL: https://theapp.your.domain.com/mycanvas.php
      * Access Method: Signed Request (POST)
  1. [Setup]>[Administer]>[Manage Apps]>[Connected Apps]>[*your app name*] and edit as follows:
    * [Profiles]>[Manage Profiles] and choose profiles to allow to use the app.
  1. [Setup]>[Administer]>[Manage Apps]>[Connected Apps]>[edit (*your app name*)] and edit as follows:
    * OAuth policies
      * Permitted Users: Admin approved users and pre-authorized


# To Increase Security

To use safely, we recommend that you set the optional security configrations.

## Don't set 'OWNERID'.

You can allow accessing the app without the user identifier parameter by setting 'OWNERID'.
```
https://theapp.your.domain.com/private/secret.php
```
It is permanent url and you can't change it.  
By leaking of the url, you can't continue the service if you want to keep the calendar secret.

## Remove [/private/secret.php](https://github.com/shellyln/iCalForce/blob/master/iCalForce/public_html/private/secret.php) from deploying image.

We don't recomend 'u=15charsCaseSensitiveUserId' style url.  
By removing 'secret.php', you can forbid it.

## White-List

You can restrict access the app to listed users.

### Generate white-list automatically
  1. Add custom checkbox field **"UseICalForce__c"** to **User** standard object.
  
  1. Set **UseICalForce__c** = true if the user is permitted to use this app.

  1. Edit **iCalForce.repo/iCalForce/tools/run-create-whitelist.sh**  
     and overwrite USERNAME, PASSWORD.
     ```bash
     #!/bin/bash
     env \
       USERNAME='alice@example.com' \
       PASSWORD='passSecuritytoken' \
       php create-whitelist.php > ../config/whitelist.php
     ```

  1. Run command
     ```bash
     $ cd iCalForce.repo/iCalForce/tools
     $ bash ./run-create-whitelist.sh
     ```

  1. View whitelist.php

  1. and access the url

    ```
    https://theapp.your.domain.com/calendar.php?t=userPublicToken
    ```
or
    ```
    https://theapp.your.domain.com/private/secret.php?u=15charsCaseSensitiveUserId
    ```

**userPublicToken** is written in **whitelist.php** as follows:
```php
  '15charsCaseSensitiveUserId' => array('pub-token' => 'userPublicToken'),
```
if you write a line of whitelist as follows:
```php
  '15charsCaseSensitiveUserId' => array('pub-token' => 'userPublicToken', 'allow-detail' => true),
```
and access the following url
```
https://theapp.your.domain.com/calendar.php?t=userPublicToken&m=1
```
the calendar will include events' description.

**We recomend using 't=userPublicToken' style url.**  
'u=15charsCaseSensitiveUserId' style url is permanent url and you can't change it.  
By leaking of the url, you can't continue the service if you want to keep the calendar secret.

### Reset the user public token
  1. Run command
     ```bash
     $ cd iCalForce.repo/iCalForce/tools
     $ env \
       USERNAME='alice@example.com' \
       PASSWORD='passSecuritytoken' \
       php update-whitelist-pubtoken.php 15charsCaseSensitiveUserId > ../config/whitelist.php.new
     ```

  1. Replace the whitelist
     ```bash
     $ cd ../config
     $ mv whitelist.php whitelist.php.old
     $ mv whitelist.php.new whitelist.php
     ```

  1. View whitelist.php

  1. and access the url

## Salesforce sharing configurations
We recommend that you create a **dedicated account** for this app  
and configure sharing and access control settings.

  * define dedicated account.
  * define dedicated user profile for the dedicated account.
  * allow api access.
  * limit accessible object by using user profile.  
    you should
    * allow **"View all data"** about User, Event.
    * disallow **"view"** about the others.
    * disallow **"edit/deelete"** all objects.
  * limit accessible fields by defining **field-level security**.

