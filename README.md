###### Raymond Gyabeng & Emily Peri
# Project 3
#### Weaponize Vulnerabilty 
To post on Dirks’ wall, we made use of the `SESSION_ID` vulnerability from Test 5. We forced the server to retrieve Dirks’ username from the database rather than xoxogg by changing the cookie to to `SESSION_ID=0' OR username='dirks`. When the server looks-up the `SESSION_ID` in the database, it runs the code `database.fetchone("SELECT username FROM sessions WHERE id='{}';".format(session))`. With the malformed cookie, the SQL query becomes `SELECT username FROM sessions WHERE id='0' OR username='dirks';` Then, we sent a POST request to http://127.0.0.1:5000/post with the message we wanted on Dirks’ wall in the request body. This attack could have been prevented if the server sanitized the cookie before using to it look up the username in the database 
#### Vulnerability Writeup
##### Test 1
Test 1 detected a reflected cross-site-scripting vulnerability. In the server.py module, the line `@app.route(‘/wall/<other_username>’)` told the server to call the function `wall` whenever a client requested the page http://127.0.0.1:5000/wall/<other_username>. If `other_username` was not found, an error page would notify the client. However, this error page inserted `other_username` into the html without checking it for HTML or javascript. Thus, a malicous user could create a reflected xss attack by inserting javascript as <other_username>. We used the URL http://127.0.0.1:5000/wall/%3CIMG%20src=%22dirks.jpg%22%20onmouseover=%22alert(123)%22%3E which inserted a (broken) image tag with onmouseover event: `<IMG src="dirks.jpg" onmouseover="alert(123)">`. To avoid this vulnterabilty, the server could only accept `other_username` that contains alphabet letters and numbers. Furthermore, the website does not even need to insert the username into the error page, it could simply alert the client that the username does not exist.
##### Test 5
Test 5 checked for a change in the cookie. Using developer tools, we were able to change the cookie value `SESSION_ID`, and in fact could use this to inject SQL since the cookie values were not sanitized by the server. The server code `request.cookies.get('SESSION_ID', '')` and the lack of validation prior to runnning `database.fetchone("SELECT username FROM sessions WHERE id='{}';".format(session))` allows an attacker to search for the session id of their choosing.  We changed the cookie to `SESSION_ID=0' OR username='dirks`. This problem could be mitigated by santizing the cookie before looking it up in the database.
##### Test 6
The vulnerability detected test 6 was a javascript event injection into the image tag for the user’s avatar. We were able to use the URL https://upload.wikimedia.org/wikipedia/commons/thumb/2/2b/WelshCorgi.jpeg/440px-WelshCorgi.jpeg\" onmouseover=alert(document.cookie) to inject a onmouseover event that would persist on the malicious user’s wall. Then when a victim vists the attacker's wall, they would forced into performing the action designated by the onmouseover event. This attack was possible due to the limitation of the `escape_html` function, and the ability to insert a double quotation character into the avatar URL. If quotation character from the avatar input was encoded then it would not end the `src` flag inside the image tag, and prevent the malicous user from adding an additonal event.
#### Other Issues
##### Insecure Transmission
Because the site does not use HTTPS, the connection in not secure and all information transmitted in unencrypted, including the session cookies. In the header response, the secure flag is not set for the cookies. According to OWASP, 
> The purpose of the secure flag is to prevent cookies from being observed by unauthorized parties due to the transmission of a the cookie in clear text...the browser will not send a cookie with the secure flag set over an unencrypted HTTP request.

Encrypting the session cookie could prevent an attacker from being able to read a users cookie when they are usign a unsecure network such as a public wifi network.
##### CSRF Vulnerabilities
There are two main problems with the current CSRF prevention in the server code. First, the function it uses to check the request headers, `csrf_protect`, is not called when a client accesses the login page. Therefore, an attacker could create a POST request from a malicous website and trick a victim into the attacker's account. Then the attacker could steal senstive information that the victim accidently inputs into the attacker's account. 

The second problem is that an attacker can circumvent the `csrf_protect` function by redirecting the vicitm first to Snapitterbook Home Page and then to a subdirectory page, using javascript on a malicious website. Furthermore, some indivduals prefer to strip the referer tags from the request headers to prevent websites from tracking their online behavior. One soultion is to require CSRF token with any POST, retrieved from a GET request to that same web page. Since an attacker will be unable to guess or access this token, they will be unable to force a victim into making a POST request.

##### Password Dictionary Attack
According to OWASP, a server should "design password storage assuming eventual compromise." Thus, we can assume that an attacker may eventually gain access to the server's database, and be able to view the hashed passwords. In order to prevent the attacker from quickly searching for commonly-used hashed passwords, and reverse engineer a user's password, the server could hash the each password with random salt. This will slow the attacker down and hopefully give the server managers time to detect the attack. 
