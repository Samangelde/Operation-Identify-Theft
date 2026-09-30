# Investigating an Identity Attack in Entra ID
Reconstructed a five-stage OAuth consent-phishing kill chain in a live Azure tenant through forensic analysis of two linked app registrations."

What I learned:
  
  • I learned why this attack works the way it did, why it was used and the differences between regular credential phishing and 0Auth phishing. Regular cred phishing has to deal with device compliance, location rules and MFA. 0Auth phishing bypasses all of it because its using the victims authenticated token with a trusted device. The resulting 0Auth2PermissionGrant does not get removed by a password reset, revoking session or by enforcing MFA.
  
  • Investigating the different tabs within the Applications registration and walking through the different items to look for an investigate. Once you have an understanding of whats an anomaly and whats a baseline you can then move through and document and dig deeper on what happened.
  
  • When bringing up a “confused Deputy” attack I had never heard of this specifically. Once you see it in action you get a better grasp of what it is or could be to identify it later.

  
  • I didn’t fully understand how the persistence was created and the foot hold maintained till I saw that the attacker created the scope in the legacy app to maintain access if the secret was rotated.
  
  • What I would do differently is since this is a legacy app and Carl shouldn’t of been able to change something without a red flag going up somewhere is monitoring. I feel a change request should have had to of been done for anything to be fully approved for it to go live. Or monitoring based on anomalous events since this would fall outside of that baseline to then be contained and eradicated.



Environment: Live-Multi-User Azure Training tenant, Reader Access


What broke / what surprised me:
What was new for me was navigating the "Expose API" tab and understanding what I was actually doing. I was surprised that a standard user like Carl was able to register an app and that having permissions to it wasn’t logged. Once it clicked everything made sense. I really enjoyed using Cyberchef a tool I have gotten familiar with during my time with the NCL. 

Findings and recommendations:
Carls credentials were stolen and then used as a consent phishing attack which led to his permissions for a legacy app to be abused. The attacker created a persistence in the API by changing the expiration to near infinite amount of time, creating a new rogue app and making it an owner in the legacy app, used the “Expose API” Tab to create a new scope that let the attacker maintain access with their rogue app even if secrets were rotated. Finally, the attacker created a malicious Redirect URI which when accessed would have the victim enter their credentials, receive an authentication token and then kick them back to the main login. 
Revoke the client secret, remove the rogue service principal from Owners, delete the custom exposed API scope, revoke the 0Auth2PermissionGrant explicitly>permissions>User Consent> 0Auth2 Permissions > Locate the one you need > Revoke,, remove the redirect URI, disable default user app registration, conduct an audit of any app registration list with owners, alert on new client secrets and new redirect URIS.

Scenario:
Carl from accounting was phished by a false login page. The attackers were able to extract his login credentials and even completed MFA. Carl was an owner to an enterprise app that was legacy based and the attackers were able to gain access to it. Turned the permissions into tenant wide Directory.Read.All and User.ReadWrite.All, they set the client secret to an impossible time frame of expiration. They also created a second owner to anchor and used that as a persistence in case the old credentials rotated out they could get new credentials. Finally, they exposed an API with a redirect URL that took the people interacting it to an authentication portal for the attacker to steal more credentials.

Investigation:
The investigation started by searching app registration in the top<img width="953" height="667" alt="Step 1" src="https://github.com/user-attachments/assets/291a0f49-555a-49df-ba74-76ffafda3174" />
Then you go to all applications tab towards the left, look for Mad-Hat-Legacy-Sync-Service <img width="584" height="619" alt="Step 1 2" src="https://github.com/user-attachments/assets/ad529769-e43a-4f1f-9370-e34a4b77aa9f" />
<img width="586" height="657" alt="step 1 3" src="https://github.com/user-attachments/assets/ed84c13a-219a-455c-95fc-f7ce6e6370ac" />

Open branding and properties and within the notes value there is something suspicious<img width="584" height="549" alt="Step 2" src="https://github.com/user-attachments/assets/fbaec423-5351-4287-aa88-d19f43bd0430" />

In order for the attacker to escalate his privileges they used Carl’s ownership to generate a new client secret. They instantly elevated their standard user to a highly privileged account granting direct access to the API Permissions. 
On the left under “Manage” you’ll see “Certificates and Secrets” Click the Client secrets tab and you’ll the time of expiration.. The expiration seems like its for an extreme amount of time<img width="584" height="389" alt="Step 3" src="https://github.com/user-attachments/assets/ae5b3c59-a245-4c6c-a4b6-5775cf4de3d7" />

In order to maintain persistence the attacker didn’t rely on the secret not being rotated and instead created a new app registration Mad-Hat-Labs-App. They added the service principal to “Owners” of the Mad-Hat-Legacy-Sync-Service. This let them be granted owner permissions over the app they registered. This is due to Carl being an admin/owner on the legacy app.If you look under “API Permissions” you’ll see what was granted to the app<img width="955" height="775" alt="step 4" src="https://github.com/user-attachments/assets/e728f8f0-cd12-47cf-ac28-704f38225424" />

Navigate to Mad-Hat-Labs-App under “Branding and properties” There is a note<img width="589" height="381" alt="Step 5" src="https://github.com/user-attachments/assets/47eca132-26da-4721-8eb7-1c791a25ed42" />

The attacker using Carl’s access used the “Expose and API” tab within the Mad-Hat-Legacy-Sync-service app. If their client secret gets removed from the legacy app the customer can use the custom api to launch another attack using the Mad-Hat-Labs-App. Access the Mad-Hat-Legacy-Sync-Service app and on the left under manage you’ll see “Expose an API” Click this. Within the custom scope name you’ll see the mask the attacker used under “User Content Display Name"

<img width="583" height="325" alt="step 6" src="https://github.com/user-attachments/assets/485a2dde-9a61-440e-8aa2-420fd3f3ee92" />

Now the attacker has control of the rogue app and the legacy app. They are able to setup a trap to collect tokens from users. The attacker has changed the redirect URI in its Authentication tab where when the customer clicks into it, they are then taken to a login that once they login and provide a code it kicks them back to the beginning of the portal as if they had a failed login without any warning. An example from the lesson of a crated URI is  below




https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/authorize?
client_id={ROGUE-APP-CLIENT-ID}
&response_type=code
&redirect_uri={ATTACKER-TRAP-URL}
&scope=api://{LEGACY-APP-CLIENT-ID}/Legacy.Sync

When a user clicks into this they are then hit with the attack described above. The rogue app gets a token for the exposed API. Acting as the user it does not give the legacy apps permissions.  It can allow for the attacker to act as the user on the back-end to utilize a sync privileged API Permissions. This is called a “Confused Deputy” attack which is when a trusted tool executes a request it shouldn’t. Using the link provided it leads you into the rogue app which will ask for your login creds. When you input them it brings you to another portal. If you decode the URL using something like cyber chef you’ll receive the flag<img width="586" height="555" alt="Step 7" src="https://github.com/user-attachments/assets/5ba3c0ba-f643-49bc-b693-688bd4e27baa" />

