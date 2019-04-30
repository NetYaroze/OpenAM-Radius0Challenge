### OPENAM-8704 -Have option to avoid extra "0" radius challenge in Radius Server

In a default  Radius Client --> LDAP ---> OATH scenario, you may be presented with a challenge that breaks the login for the RADIUS client:

> amRadiusServer:04/25/2019 04:59:06:939 PM MEST:
> Thread[RadiusRequestHandler-0,5,main]:
> TransactionId[635b6afd-7fc8-4ff1-8338-3d01a2fd06e8]   WARNING:   
> Packet to EPlex30:     ACCESS_CHALLENGE [1]  
>     - STATE : 3067bbf8-150c-47e4-9375-e745b78e157f  
>     - REPLY_MESSAGE : ForgeRock Authenticator (OATH)  (Default is 0.)   0 = Register device

The workaround should really be a custom module since you probably do not want to mess about with the default Authentication module anyway, but a workaround nonetheless (with possible temperamental results depending on your use case). 

My use case in this example assumes that devices do not need to register via RADIUS (in a browser / somewhere else etc). 

So Radius Client --> LDAP ---> OATH ---> Enter TOTP ---> AuthN.

Say we have a chain like this (it could be Required aswell):
![enter image description here](https://lh3.googleusercontent.com/2ik1ri3Dyhh0isO7GqUjmaaSKPnWhW2GnownsHfE6iRLpZmTR4aYpaT8SjBi_3S2mmmFUwM4YWyy) 

With Settings:

![enter image description here](https://lh3.googleusercontent.com/L4T0216QyqB1bz5taEh9H4c-06kRaFj-d3WragF7BAFtxQYGRl88TG3NcsYpWBedal8LShwFUVcg)

We setup a otpuser and ensure that this has the OATH setup (before all this)

    $ldapsearch -h localhost -p 10389 -D "cn=Directory Manager" -b
    "ou=people,dc=openam,dc=forgerock,dc=org" "(uid=radiuser)" '*'
    dn: uid=radiuser,ou=people,dc=openam,dc=forgerock,dc=org
    objectClass: iplanet-am-managed-person
    objectClass: inetuser
    objectClass: sunFederationManagerDataStore
    objectClass: sunFMSAML2NameIdentifier
    objectClass: inetorgperson
    objectClass: sunIdentityServerLibertyPPService
    objectClass: devicePrintProfilesContainer
    objectClass: iplanet-am-user-service
    objectClass: iPlanetPreferences
    objectClass: pushDeviceProfilesContainer
    objectClass: forgerock-am-dashboard-service
    objectClass: organizationalperson
    objectClass: top
    objectClass: kbaInfoContainer
    objectClass: person
    objectClass: sunAMAuthAccountLockout
    objectClass: oathDeviceProfilesContainer
    objectClass: webauthnDeviceProfilesContainer
    objectClass: iplanet-am-auth-configuration-service
    cn: radiuser
    employeeNumber: 0
    inetUserStatus: Active
    iplanet-am-user-auth-config: [Empty]
    mail: radiuser@example.com
    oath2faEnabled: 2
    oathDeviceProfiles: { "uuid": "d569bb53-ea95-42c1-b5e1-3ddac254673d", "recoveryCodes": [ ],
    "sharedSecret": "D04609C289BB756E42751340F95FEB43", "deviceName": "OATH Device",
    "lastLogin": 0, "counter": 3, "checksumDigit": false, "truncationOffset": 0, "clockDriftSeconds":
    0 }
    sn: radiuser	
    uid: radiuser
    userPassword: {CLEAR}something

The default XML provided in this repo, *should* work as a base example, depending on your use case and whether it matches mine above. 

The AuthenticatorOATH.xml provided needs to be added to your deployments directory like so:

    /home/fr/apache/webapps/openam/config/auth/default_en

Given its pretty minimal, it's just there to show the changes so you can just append it (or git clone it).

You should now see things working (RADIUS client is there for you to use in AM to test):

    fr@cveam5:~/apache/webapps/openam/WEB-INF/lib$ java -jar openam-radius-server-14.0.0.jar
    ? Username: radiuser
    ? Password: cangetin
    
    Packet To 127.0.0.1:1812
      ACCESS_REQUEST [1]
        - USER_NAME : radiuser2
        - USER_PASSWORD : *******
        - NAS_IP_ADDRESS : cveam5.fr.local/127.0.0.1
        - NAS_PORT : 0
    
    Packet From 127.0.0.1:1812
      ACCESS_CHALLENGE [1]
        - STATE : 820c67d9-9981-42c0-9c1c-aff57d7e60d9
        - REPLY_MESSAGE : ForgeRock Authenticator (OATH) Enter verification code:
    
    ---> ForgeRock Authenticator (OATH) Enter verification code:
    ? Answer: 570032
    
    Packet To 127.0.0.1:1812
      ACCESS_REQUEST [2]
        - USER_NAME : radiuser2
        - USER_PASSWORD : *******
        - NAS_IP_ADDRESS : cveam5.fr.local/127.0.0.1
        - NAS_PORT : 0
        - STATE : 820c67d9-9981-42c0-9c1c-aff57d7e60d9
    
    Packet From 127.0.0.1:1812
      ACCESS_ACCEPT [2]
    
    ---> SUCCESS! You've Authenticated!

## Bonus Round
If you want to go further with this (i.e on step 7). 

You can have a look at the changes to default AuthenticatorOATH.xml like so. 

    diff -u
    orig/ws-650/app/webapps/openam/config/auth/default_en/AuthenticatorOATH.xml
    new/ws-650x/app/webapps/openam/config/auth/default_en/AuthenticatorOATH.xml
    --- orig/ws-650/app/webapps/openam/config/auth/default_en/AuthenticatorOATH.xml
    2019-01-15 06:38:00.000000000 +0800
    +++ new/ws-650x/app/webapps/openam/config/auth/default_en/AuthenticatorOATH.xml
    2019-03-02 16:34:18.269446008 +0800
    @@ -39,10 +39,14 @@
    </ConfirmationCallback>
    </Callbacks>
    <!-- For when we're not optional and a device is registered -->
    +<!--
    <Callbacks length="2" order="4" timeout="120" header="#REPLACE#">
    +-->
    + <Callbacks length="1" order="4" timeout="120" header="#REPLACE#">
    <NameCallback>
    <Prompt>Enter verification code:</Prompt>
    </NameCallback>
    +<!--
    <ConfirmationCallback>
    <OptionValues>
    <OptionValue>
    @@ -50,6 +54,7 @@
    </OptionValue>
    </OptionValues>
    </ConfirmationCallback>
    +-->
    </Callbacks>
    <!-- For registration -->
    <Callbacks length="3" order="5" timeout="120" header="Register a device">
    @@ -81,10 +86,14 @@
    </ConfirmationCallback>
    </Callbacks>
    <!-- For when we're optional, but we just generated a device -->
    +<!--    
    <Callbacks length="2" order="7" timeout="120" header="#REPLACE#">
    +-->
    + <Callbacks length="1" order="7" timeout="120" header="#REPLACE#">    
    <NameCallback>    
    <Prompt>Enter verification code:</Prompt>    
    </NameCallback>    
    +<!--
    <ConfirmationCallback>   
    <OptionValues>
    <OptionValue>
    @@ -95,6 +104,7 @@
    </OptionValue>
    </OptionValues>
    </ConfirmationCallback>
    +-->
    </Callbacks>
    <!-- Display recovery code -->
    <Callbacks length="11" order="8" timeout="120" header="ForgeRock
    Authenticator (OATH) Recovery Codes">
**NB note the above was done in an older version (i changed versions as my instance crashed, hence the difference in XML files.

Now if you do access this chain:

    amAuth:03/02/2019 04:39:45:713 PM SGT: Thread[http-nio-8080-exec-3,5,main]:
    TransactionId[d1e7d50e-5225-4765-8870-8f4cd609790f-3763]
    Exception
    javax.security.auth.login.LoginException: java.lang.ArrayIndexOutOfBoundsException: 1
    at
    org.forgerock.openam.authentication.modules.fr.oath.AuthenticatorOATH.process(AuthenticatorO
    ATH.java:323)
    at
    com.sun.identity.authentication.spi.AMLoginModule.wrapProcess(AMLoginModule.java:1091)
    at com.sun.identity.authentication.spi.AMLoginModule.login(AMLoginModule.java:1289)
    at sun.reflect.GeneratedMethodAccessor71.invoke(Unknown Source)
    at
    sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at com.sun.identity.authentication.jaas.LoginContext.invoke(LoginContext.java:218)
    at com.sun.identity.authentication.jaas.LoginContext.login(LoginContext.java:126)
    at
    com.sun.identity.authentication.service.AMLoginContext.runLogin(AMLoginContext.java:512)
    at
    com.sun.identity.authentication.server.AuthContextLocal.submitRequirements(AuthContextLocal.j
    ava:586)

The code for the original XML do insist there is 2 callbacks. Hence it cannot take anything less. This is the callback at order=”7” where this is asking for an OPTIONAL step. 

    case LOGIN_OPT_DEVICE:
    selectedIndex = ((ConfirmationCallback) callbacks[1]).getSelectedIndex();
    if (selectedIndex == OPT_DEVICE_SKIP_INDEX) {
    realmService.setUserSkip(id, SkipSetting.SKIPPABLE);
    realmService.removeAllUserDevices(id); //user backed out of saving device
    return ISAuthConstants.LOGIN_SUCCEED;
    }
    
    //fall through
    
    case LOGIN_SAVED_DEVICE:


One can always extend this to check if the (callbacks.length > 1) before testing it and if not
fallthrough (as there is no option).  This will settle UI stage “7”. 

However, if you want to trigger UI at stage=”4”, just go to the Realm’s authentication setting and set “Two Factor Authentication Mandatory” to skip this. This can be easily tested using XUI itself, and you shouldn't see the extra button anymore. 
