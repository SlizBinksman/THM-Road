# THM_Road
A write up on the THM box 'Road'

# Credits
Thank you for providing this box!

TryHackMe Room:  
https://tryhackme.com/room/road  

THM Room Author:  
https://www.tryhackme.com/p/StillNoob

Resources:  
https://www.hackingarticles.in/linux-privilege-escalation-using-ld_preload/ (LD_Preload Privilege Escalation)  
https://wiki.thehacker.nz/docs/thm-writeups/road-medium/ (For Privilege Escalation)  
https://www.freedesktop.org/software/polkit/docs/0.105/pkexec.1.html  
https://github.com/NixOS/nixpkgs/issues/18012  

# Write Up

[!] Note: After getting initial access to the server, I got stuck for a while and I had to use the guide listed above under Resources to figure out how to escalate privileges to root. In my opinion, It's better to look something up after a certain period of time and use that as a learning experience for the next challenge. 

Lets do some recon on the listening services. I choose rustscan for this purpose. 

![image](https://user-images.githubusercontent.com/90923369/144649993-4cfd615a-aea9-4deb-801e-e472d82f25a2.png)

So we have a pretty small attack surface here. We are limited to SSH and HTTP. Lets visit the webpage and see what comes up!

![image](https://user-images.githubusercontent.com/90923369/144650354-fb1b1402-1bbf-47b7-a4ef-9c5e832150f1.png)

Here we have what looks to be a business webpage. Lets poke around and see if we can find anything that could help us dig deeper into the system.

![image](https://user-images.githubusercontent.com/90923369/144650619-553d4914-c976-4049-ba49-7e87b1950744.png)

So if we click on the `Merchant Central` Button in the top right corner of the home page, we are presented with a login. We dont have any credentials however we can register an account. Lets make an account and see what we can find.

![image](https://user-images.githubusercontent.com/90923369/144650945-52a1eec7-b61f-45b3-83c0-0a0805fddc3f.png)

We register an account and are now provided with this panel. After looking around, i only found two different functions on the panel. One is a user password reset and the other is a page that allows you to edit your profile & upload a photo. All of the other links redirect back to `index.php` which is the main panel we are presented with after authenticating.

![image](https://user-images.githubusercontent.com/90923369/144651659-2a939a91-1879-4c66-ba11-6c9106fb3c47.png)

If we go to page that allows us to edit our profile information and photo, we can find the administrator username `admin@sky.thm`. Lets see if we can change the password with the reset user function. 

![image](https://user-images.githubusercontent.com/90923369/144652013-9036bd65-2a32-4412-bc79-3095917b328b.png)

When we land at the reset user page, we are unable to change our username in the user input box. Lets fire up burpsuite and intercept the request with the proxy function. We may be able to change the username after intercepting the web request.

![image](https://user-images.githubusercontent.com/90923369/144652491-36ddf0d6-28de-4b32-8066-e8964789c7cc.png)
![image](https://user-images.githubusercontent.com/90923369/144652554-bea706fb-2745-4fb5-8823-b1cfe78e888e.png)

By capturing the request with burpsuite, we are able to change the username to the admin username we found in the profile page. When we forward the request to the server, we can see that it says password changed successfully. Lets try and log in with the admin credentials and the password that we switched over to.

![image](https://user-images.githubusercontent.com/90923369/144654908-e026bce9-daa5-4773-836d-e606095ca2c3.png)

BOOM! Our attack was successful. We have admin access on this web application. The next step is to now go back to the profile picture upload function and see if we can upload a malicious file to gain a shell on the system. 

![image](https://user-images.githubusercontent.com/90923369/144653102-e269b3df-1034-4b7e-b9b3-caa0fb47bfa4.png)
![image](https://user-images.githubusercontent.com/90923369/144653266-2a3bcb09-d4c5-4b6f-980c-f231e2121bdd.png)

I have chosen Pentest Monkeys PHP shell. We can see that we get a response saying image saved. This means that our malicious file has been saved to the system. Our next step is to figure out where this file has been saved so we can execute it.

![image](https://user-images.githubusercontent.com/90923369/144654001-c7645471-588b-405e-8eaa-a562aa2bfa49.png)

It took me a decent amount of time with a few different rabbit holes however i was able to find a promising directory listed in the target tab in burp. The directory `profileimages` in the `v2` directory seems like it should hold our `php-reverse-shell.php` file. Lets navigate to it in the browser.

![image](https://user-images.githubusercontent.com/90923369/144654372-178e6109-b755-46c7-a278-ff5aa4c85664.png)

Oof....We'll have to manually type out the file name in the address bar because we are not allowed to see the files stored in the directory. Lets first open a netcat listener.

![image](https://user-images.githubusercontent.com/90923369/144654678-10b16079-16b7-4657-a5c2-86bbed010dd6.png)

YEAAAHH BOIIII. We got a shell on the system. Lets upgrade the shell so we can see what we're doing more clearly.

![image](https://user-images.githubusercontent.com/90923369/147796250-9cbf5326-0a22-4b82-bf13-98e5c74f0cb0.png)

Much better. The first thing i checked was the `/etc/passwd` file to see what other users were on the machine.

![image](https://user-images.githubusercontent.com/90923369/147796339-58cb0883-6ddc-46f4-ae4a-38b224e66eac.png)

Pretty interesting. We have a user named webdeveloper and it looks like there might be a database in this system since mongodb is in there. To open MongoDB you just have to run the command `mongo`. Lets try it and see if we get a mongo shell.

![image](https://user-images.githubusercontent.com/90923369/147796465-038489a7-afbc-4f62-b0e7-6775daa76d67.png)

Boom. We're in a mongo shell. Lets run `show dbs` from the help menu and see if there's anything interesting.

![image](https://user-images.githubusercontent.com/90923369/147796503-f7ee1218-faab-4669-a4fc-7990f9a7f6cd.png)

We have 4 different databases here. To save time, there is nothing in the first one. The second one: `backup`, is what we're after.

![image](https://user-images.githubusercontent.com/90923369/147796557-7fc496fd-b3f2-4a79-b479-8dccfcc9e74e.png)

To access the the database, we use the command `use {database}`. Once we have the object, we can use the command `show collections` to show whats inside of it. 
When we do that, we can see that there is a collection called `user`. By using the command `db.user.find()`, we can get the information contained inside of it. That information would be a password & username. Lets try switching to that user.

![image](https://user-images.githubusercontent.com/90923369/147796790-1606a2c6-60de-4960-bbbb-3598ffcb6369.png)

Awesome! before i forget....

![image](https://user-images.githubusercontent.com/90923369/147796876-bb9decf0-3179-4e12-85be-185d0e975dc3.png)

The user flag.

The next thing to do would be checking the suid binarys with `find / -perm -u=s -type f 2>/dev/null`.

![image](https://user-images.githubusercontent.com/90923369/147855720-368fb155-15d5-4d4b-8e40-767ad0c5d0d9.png)

You can see that we have `/usr/bin/pkexec` listed. `pkexec` is a program that allows the user to execute a program as another user. If no user passed into the command, it will run the program as root by default.

![image](https://user-images.githubusercontent.com/90923369/147855911-4107ae80-152a-4ed1-ac02-515c562e0393.png)

When we try to spawn a root bash shell with pkexec, the authentication fails and we're handed an error which i have highlighted in green.

![image](https://user-images.githubusercontent.com/90923369/147856017-27f19c49-1e67-4425-8d16-2164d8de1122.png)

If we take the error and punch it into a search engine, the first result we get is a Github bug report.

![image](https://user-images.githubusercontent.com/90923369/147856259-18dce73d-acc8-4d3a-aa48-66f3dd249e6d.png)
![image](https://user-images.githubusercontent.com/90923369/147856275-1af1aa88-5439-4d0d-b20a-46c4e3a9906b.png)

There is a bug in the authentication when running pkexec without a gui. To get around this, we need to open another terminal and connect through ssh. When we have 2 sessions, we can use them like this.

![image](https://user-images.githubusercontent.com/90923369/147856785-b2f81236-3464-4e0a-8896-605aaaef90a7.png)

The first step is to make sure that both of your sessions are running on the same account. Next, we need to find our pid that we live in by using `echo $$`. In the other session, we run `pkttyagent --process {PID}` so we can act as an authentication agent. We then go back to the first terminal and run `pkexec "/bin/bash"`. It will run as root by default because no user was passed into the command. Back on the 2nd terminal, we will need to authenticate with a password. When we're authenticated, the first terminal will now spawn a bash shell as root! Before i forget.....

![image](https://user-images.githubusercontent.com/90923369/147856934-fa870dd5-06cc-44e7-ac43-ca4241e43356.png)

# Second Escalation Vector (LD_Preload)

While looking for a way to escalate privileges, I saw that other people were able to escalate privileges with some C code because of a vulnerability with the enviroment variable `LD_Preload`. 

![image](https://user-images.githubusercontent.com/90923369/148002979-558ea44e-f88e-43ce-9136-6333cc8ac842.png)

When we run `sudo -l`, there is a return parameter all the way to the right called `env_keep`. The env_keep parameter specifies an environment variable that will be preserved when the command is run with sudo. The LD_Preload variable allows the user to load shared librarys before other librarys are loaded in a program. For example, If you make your own `cout` function in C, you can overide the standard cout function with your function. Lets check if `gcc` is installed on the system.

![image](https://user-images.githubusercontent.com/90923369/148003758-669538fd-e8dd-49bb-bae4-723d12a2e136.png)

gcc is installed on the system. Lets open up a text editor. I prefer nano.

![image](https://user-images.githubusercontent.com/90923369/148470207-90518a6d-ff08-443b-bcce-805d1e6383fa.png)

This code will allow us to set the process to root privilege and then execute an sh shell.

![image](https://user-images.githubusercontent.com/90923369/148469900-64ee045d-a754-4288-9481-210b4804c08b.png)

We compile the code with gcc. The `.so` file extention is the linux equivilant of the `.dll` file extention. Since we have LD_Preload as an environment variable, we can run `sudo LD_PRELOAD=/home/webdeveloper/shell.so /usr/bin/sky_backup_utility` which will execute our imported library and give us a root shell. This will be the first thing that get executed by the sky_backup_utility program. After we press `control C`, We will kill our root shell and then backup program will perform the backup operation. This is because our root shell lived inside of the program and was the first thing to be executed since we preloaded the library.
