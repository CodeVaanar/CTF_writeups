# Interceptor - TryHackMe Writeup 

started with Nmap port scan but nothing notable was found except we have a web page we can check out . that web page was a login page wanted to try sql injectiona nd did but it failed .

so went back to the terminal and tried Gobuster directory enumeration with extension (-x flag).

```
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php.bak,sql,bak,js.bak,log,tar.gz,zip
```
<img width="586" height="171" alt="image" src="https://github.com/user-attachments/assets/d24b20e9-32c1-4a99-8d7a-f586ce952de7" />

when *login.php.bak* was used in the web browser it immediately downloaded it but my attack machine didn't had anything to download it so i used curl instead 

```
	Curl http://10/49.170.91/login.php.bak
```

and this gave the developer's note.

<img width="858" height="312" alt="image" src="https://github.com/user-attachments/assets/3b546cc2-cb58-4404-8147-761feed16bff" />

username was clearly given but not password, but it clearly was mentioned that we only need the correct year - so i randomly tried 2026 and it worked but we could also have created a wordlist for the years but in that case we might run out of chances to use any new year , so in that case we would have to go to storage and then delete the cookie. and try again until the correct one was found 

anyhow we have the password - MediaHub2026.

and we are logged in but we have a OTP verification.

Remember we do not have the access to the admin email to be able to get the OtP so we will have to resort to a different method 

First checked the page source ->This specific piece of code indicates that the frontend is communicating with a backend endpoint and expecting a JSON response. It checks whether the backend reports the operation as successful (data.ok), displays either the returned error or success message to the user, and, on success, redirects the user to dashboard.php after 700 ms. If something unexpected goes wrong, the catch block displays a generic error and re-enables the button.

<img width="802" height="305" alt="image" src="https://github.com/user-attachments/assets/43d856ba-4a06-4714-a155-84a7430243d7" />


now i submitted a false otp - caught it with burp 

So if we change the request's name from 'otp' to 'is_verified' and 123456 to True -> we should be able to bypass the OTP

But how ?
The idea is that if the backend blindly accepts and processes the client-controlled `is_verified` parameter, we can make the server believe that the user has already been verified without providing the correct OTP. 
<img width="990" height="294" alt="image" src="https://github.com/user-attachments/assets/0ae13924-972e-49ba-bd5d-2fc245fe6e2f" />

This is essentially a parameter tampering / mass-assignment style vulnerability: a security-sensitive value that should be controlled by the server is being accepted from the client.

and we have our first flag 

<img width="1279" height="425" alt="image" src="https://github.com/user-attachments/assets/e5c7ca79-5075-4c0e-b355-ff842c55f1e9" />

## second flag 

<img width="758" height="222" alt="image" src="https://github.com/user-attachments/assets/69f5caee-e3e6-4c98-9612-39a4081bb502" />

we were given this so we are try and see if this will led us to our second flag 

1. we had no valid url so tried http://127.0.0.1- but it was filtered out
   <img width="517" height="249" alt="image" src="https://github.com/user-attachments/assets/bac4219d-63e3-4427-88cf-ed51787dcba1" />

2. so i tried it's short form (12.0.1) and it worked!
   <img width="695" height="367" alt="image" src="https://github.com/user-attachments/assets/3974b210-25e8-441b-aefc-fa9a00dc72d2" />

3. now that we have the local host - tried to fetch the content of the file /var/www/user.txt.
   <img width="701" height="284" alt="image" src="https://github.com/user-attachments/assets/f2f7b7fd-0636-4d7b-8508-fb16d3d02887" />

   and it returned with our second and last flag.
   <img width="522" height="198" alt="image" src="https://github.com/user-attachments/assets/5f3b9cc9-1341-417b-80e7-e57111418681" />


that's it !!!

