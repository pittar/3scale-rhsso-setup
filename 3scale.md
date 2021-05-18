# 3Scale Setup

## Install 3Scale

```
oc new-project 3scale

oc create -f <registry access secret>

oc create -f https://raw.githubusercontent.com/3scale/3scale-amp-openshift-templates/2.6.0.GA/amp/amp-eval-tech-preview.yml

# This will probably also require a param for the RWO storage class, and maybe the Postgres version.
oc new-app 3scale-api-management-eval \
    -p TENANT_NAME=dfo-dmp \
    -p WILDCARD_DOMAIN=<openshift domain or ip> \
    -p ADMIN_PASSWORD=password \
    -p MASTER_PASSWORD=password \
    -p RWX_STORAGE_CLASS=azfsc
    

# Once all the pods start, login to your new tentant admin portal at https://dfo-dmp-admin.<cluster url>
# Go to "Settings" (gear at top right) and copy your provider key, then import your first api!

# Add -k if your portal has self-signed certs.
3scale import openapi -k --default-credentials-userkey=user-key -d https://<provider key>@dfo-dmp-admin.apps-crc.testing https://petstore.swagger.io/v2/swagger.json

# Refresh your admin portal and go to "Swagger Petstore API" Service.
# Create an "Application Plan".  When you are done, make sure you publish it.
# You can then publish Publish your API service to Production.
```

## Create an Application.

In your admin account, select "Audience" from the drop down in the top-nav and select the "Developer" account.

After that, select the "1 Application" link at the top, then "Create Application".
Pick the applicatin plan you just created and give it a name and description.
You can now test your active docs.

## Developer Portal

Under "Audience" open "Developer Portal" from the left-nav and chose "Content".  Click on the button to open your portal to the world.
Next, scroll down and select the "Visit Portal" link.

## Install Red Hat SSO

```
oc new-project rhsso

oc new-app sso73-x509-postgresql-persistent \
    -p VOLUME_CAPACITY=2Gi \
    -p SSO_ADMIN_USERNAME=admin \
    -p SSO_ADMIN_PASSWORD=password \
    -p POSTGRESQL_IMAGE_STREAM_TAG=10

# Once RH-SSO is fully started, remove the env vars for the admin user/pass.  This is so the 
# admin user/pass doesn't get re-created on every pod start.  The original settings will be
# persisted in the PostgreSQL database.

oc set env dc/sso SSO_ADMIN_USERNAME-
oc set env dc/sso SSO_ADMIN_PASSWORD-
```

## Configure Red Hat SSO with Azure AD

1. Login to RH-SSO and create a new Realm called `3Scale`
2. In the 3Scale realm, go to "Identity Providers" in the left nav and select "OpenID Connect v1.0"
3. Copy the redirect URI, you will need it for your Azure AD Appliction.
4. In Azure Portal, go to `Azure Active Diretory -> App Registrations -> New Registration`
5. Give your app a name and paste the Redirect URI into the text box near the bottom of the screen beside `Web`, then click **Register**.
6. Copy the `Application (Client) ID` and paste it into the appropriate field in Red Hat SSO for your new OpenID Connect provider.
7. Back in Azure, at the top of the screen click on `Endpoints` and copy the `OpenID Connect metadata document` link, then paste it into the `Import from URL` text boc in Red Hat SSO
8. Click the **Import** button that will appear below this text box.
9. Back in Azure, click on `Certificates and Secrets` from the left nav.  Generate a new **Client Secret** (expire never).  Copy the value and paste it into the `Client Secret` text box in Red Hat SSO.
10. In Red Hat SSO, click **Save**.

## Create a 3Scale Client

1. In your 3Scale Realm in Red Hat SSO, click on `Clients` from the left nav, then `Create Client`.
2. Chose a `Client ID` (e.g. 3scale) and make sure `openid-connect` is selected
3. Click **Save**.
4. Change the access type to `confidential`.
5. Set valid redirect urls for your admin and developer portals, for example https://dfo-dmp-admin.apps-crc.testing/* and https://dfo-dmp.apps-crc.testing/*
6. For `Web Origins` add `*`
7. Save
8. Scroll to the top and click the `Credentials` tab (which should now be there).  Copy the secret.

## Setup 3Scale admin portal to use RH SSO

1. In your 3Scale admin portal, click on the gear in the top right for `Account Settings`.
2. In the left nav, click `Users -> SSO Integrations` and select `New SSO Integration -> Red Hat Single Sign-On`.
3. Add your client (probably 3Scale) and your secret that you copied in Step 7 of the **Create a 3Scale Client** instructions.cccccckddnikfjkudhhherkbbfrvecceiltreltdduhl
4. Paste in your realm, which will look something like `https://sso-rh-sso.apps-crc.testing/auth/realms/3scale`
5. Save
6. If you have self-signed certificates, `edit` your configuration and ensure the **Do not verify SSL** checkbox is checked, then save.
7. Click the `Test Authentication Flow` link and sign-in with your Azure AD.  Upon successful authentication you will be prompted to create your account.
8. Once complete, you can "Publish" your SSO integration.
9. In order to make your new user an admin that can login, you will need to login to your **masetr** portal and approve the user.

## Setup 3Scale Developer Portal to use RH SSO

1. Login to your admin tenant.
2. At the top drop down menu select **Audience**.
3. On the left nav, click `Developer Portal -> SSO Integrations`.
4. Select `Red Hat Single Sign-On`.
5. Enter the same details for `Client ID`, `Client Secret`, and `Realm` as you did in the last set of instructions.
6. If you are using self-signed certs, click `Do not verify SSL certificate`.
7. Check `Publish`.
8. Click **Create Red Hat Single Sign-On**.

## Service Discovery

https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.6/html/admin_portal_guide/service-discovery#service-discovery-concept

## Binary build for WAR file (Tomcat 8)

```
oc new-project product-list

oc new-build jboss-webserver31-tomcat8-openshift:1.2 --name=product-list-build --binary=true

oc start-build product-list-build --from-file=app.war
```

## Active Docs

Security Definition (before final closing curly bracket):
```
  "securityDefinitions": {
    "api_key": {
      "type": "apiKey",
      "name": "user-key",
      "in": "header"
    }
  }
```

Security inside the "get" (or other method) first opening curly bracket:
```
        "security": [
          {
            "api_key": [
            ]
          }
        ]
```
