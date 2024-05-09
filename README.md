# Prompting-Problems-Official-WriteUp
The official write up for the Prompting Problems CTF Challenge
This challenge is a boot2root CTF challenge targeted at beginners and getting them to use common tools that they would need in a real world situation or other CTFs. it also encourages them to identify trends that can be used to target patterns that can be exploited; mistakes rarely happen in a vacuum.

## Gaining Access

An NMAP scan of the host show 2 open ports;

```
Host is up (0.000044s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE
631/tcp  open  ipp
5000/tcp open  upnp
```

Of those two ports, port 5000 is a custom developed web application, that when visited is called "Developing Authentication and Authorization for My Fintech Startup". There is a hint as well:
- "To obtain a token, you need to find the endpoint in the correct place. However, its important that you can't use it where you're not supposed to. The place where the administrators rest."

Basically you need to find the authentication endpoint first to start off. This can be done by running a gobuster scan of the host. The endpoint that the user's are looking for are in the common.txt wordlist but does require the html extension to be found. This can be done using any tool such as GoBuster or DirB, but once that is completed the users will find:
- http://localhost:5000/authuser.html

Accessing the endpoint return a JWT token and sets it as a cookie. The token would look something like this:
- eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.ey<Reduced>.Dfvu9tWKnGRPtfO5rAK9gszPWEp1Rxcnrj5APF3yxZM

There are 2 ways this JWT can be approaching. The first way is by simply brute forcing the symmetric key on the JWT to forge one. While it can be brute forced, its not necessarily easily as the secret key is not in rockyou and may need a ruleset to crack.

The other option is to simply changOe the values in the JWT to see if the signature is correctly validated by the server. The JWT has the following fields:
```
{
  "username": "Melio",
  "password": "password",
  "userid": "0",
  "appstate": "development",
  "isAdmin": false,
  "isGuest": true,
  "domain": "melio.com"
}
```

The important fields here to switch are isAdmin and isGuest. If both are true the user won't be able to access the endpoint because by definition an admin is not a guest. 

The user will also need to find the admin endpoint to use their forged JWT on. This endpoint isn't seriously hidden but does maintain the same pattern as the previous endpoint as it is called /admin.html. This would likely be found in the previous scan as well.

Once the users have forged the JWT and have access to the admin panel. There is an endpoint saying enter IP address. entering anything will show that it performs an nmap scan against port 80.

The users can then use a ';' to end the nmap command after an IP address and then execute their own command. For example:

localhost; <Reverse Shell>

Once a reverse shell is achieved, the user flag can be found in /var/www/html

## Privilege Escalation

The user's have access to the host as the www-data user. This user has not password and very limited privileges, enumeration will show 2 other users on the host, melio and melio-app-dev as well as a backup file for app.py call app.py.bak.

The final lines in this file contain the following infomration

if _name_ == '_main_':
    app.run(host='0.0.0.0', port=5000, debug=True)
    # ensure that you're running this as sudo to prevent authorisation issues
    # My password is Meliolikespie123

This passwords can then be used to su into the melio-app-dev user.

The final step of this challenge is to get to root. A simple sudo -l for melio-app-dev shows that they can run python as root:

User melio-app-dev may run the following commands on melio-VirtualBox:
    (ALL) NOPASSWD: /usr/bin/python3

After that its a simple matter of spawning a terminal or another reverse shell to gain root access and read the root falg in the /root endpoint.
