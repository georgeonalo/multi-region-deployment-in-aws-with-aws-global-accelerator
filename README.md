# Multi region deployment with aws global accelerator

To implement a multi region deployment, otherwise known as blue/green deployment across different AWS Regions, we make use of [Global Accelerator traffic dials to dial](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-traffic-dial.html) up or down traffic to a specific AWS Region. For each AWS Region (or endpoint group), we set a traffic dial to control the percentage of traffic that is directed to that Region.
 
 
Once you create and test the new version of your application, first set the traffic dial to 0 to cut off traffic for the green Region, the next available Region will serve the traffic that was supposed to be served by the green region. Remove the previous endpoints from the endpoint group (or set their endpoint weights to 0), add the new version of the application as endpoints to the endpoint group.


When the new endpoints are healthy, shift the traffic all at once by updating the traffic dial from 0% to 100% for the green Region and from 100% to 0% for the blue Region, or gradually increase the traffic dial for the green Region from 0% to 100%, and gradually decrease the traffic dial from 100% to 0% for the blue Region. This provides the ability to perform canary analysis where a small percentage of production traffic is introduced to the new environment.




![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/20c9cd1f-7f4c-4ab1-a96e-3ddffb7fb767)



If issues arise during the deployment, achieve rollback by updating the traffic dial to 0 for the green Region, removing the new (green) endpoints, adding the previous (blue) endpoints or setting their endpoint weights back to the original, update the traffic dial to 100% for the green Region.



Updating traffic dials can be done via [AWS Global Accelerator console](https://docs.aws.amazon.com/global-accelerator/latest/dg/getting-started.html) or the following CLI command:




```
$ aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:accelerator/1234abcd-abcd-1234-abcd-1234abcdefgh/listener/6789vxyz-vxyz-6789-vxyz-6789lmnopqrs/endpoint-group/ab88888example \
  --traffic-dial-percentage 0
```



![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/f2dce5a9-c3c8-493f-aa2c-836b61a60cca)


# Example of blue/green deployment for a multi-region application

I would like to implement a blue/green deployment for a multi-region application deployed in two AWS Regions: US-WEST-2 (Oregon) and EU-WEST-1 (Dublin). The application consists of an Application Load Balancer in each Region, acting as endpoints for our accelerator


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/ba28cedb-6fd8-465c-ad7a-69b0b6a099d5)




In this example I execute the following Bash script that uses [cURL](https://curl.se/) to simulate 100 requests to the accelerator DNS and output a count of where each request was processed:



```
$ for ((i=0;i<100;i++)); do curl http://aebd116200e8c28ad.awsglobalaccelerator.com/ --silent >> output.txt; done; cat output.txt | sort | uniq -c ; rm output.txt;
```



For more information on how Global Accelerator routes client requests, see [how AWS Global Accelerator works](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html) in the documentation.



Requests from a client who should be served by US-WEST-2 region:



```
$ for ((i=0;i<100;i++)); do curl http://aebd116200e8c28ad.awsglobalaccelerator.com/ --silent >> output.txt; done; cat output.txt | sort | uniq -c ; rm output.txt;
 100 requests processed in US-WEST-2 (BLUE Environment)
 ```
 
 
 I would like to deploy the green environment in US-WEST-2 region, for this I set the traffic dial to 0 for the endpoint group to cut off traffic for the green Region, this can be done via Global Accelerator console or via APIs:
 
 
 
 ```
 $ aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:accelerator/1234abcd-abcd-1234-abcd-1234abcdefgh/listener/6789vxyz-vxyz-6789-vxyz-6789lmnopqrs/endpoint-group/ab88888example \
  --traffic-dial-percentage 0
  ```
  
  
  
  ![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/7c51b9b2-3e67-45a2-b758-dd65b13b7ecd)



Requests from a client who should be served by US-WEST-2 region:




```
$ for ((i=0;i<100;i++)); do curl http://aebd116200e8c28ad.awsglobalaccelerator.com/ --silent >> output.txt; done; cat output.txt | sort | uniq -c ; rm output.txt;
 100 requests processed in EU-WEST-1 (BLUE Environment)
 ```
 
 
 
 All the new traffic is now served from the next available endpoint group, which is EU-WEST-1.
 
 
 
 I’ve updated my application in US-WEST-2 Region and would like to test it, I set its traffic dial to 10% – the endpoint group should handle 10% of traffic that is supposed to go in US-WEST-2
 
 
 
 ![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/d181028d-5afb-48ab-9d37-18c447f159a6)





Requests from a client who should be served by US-WEST-2 region:





```
$ for ((i=0;i<100;i++)); do  curl http://aebd116200e8c28ad.awsglobalaccelerator.com/ --silent >> output.txt; done; cat output.txt | sort | uniq -c ; rm output.txt;
  90 requests processed in EU-WEST-1 (BLUE Environment)
  10 requests processed in US-WEST-2 (GREEN Environment)
  ```
  
  
  
  
  10% of the new US-WEST-2 traffic is served from the green environment (US-WEST-2), and 90% from the blue environment (EU-WEST-1).
  
  
  Note: Traffic dials control the percentage of traffic that is directed to the group. The percentage is applied only to traffic that is already directed to the endpoint group, not to all listener traffic. If you want for example the green environment to serve 10% of all your traffic, set its traffic dial to 10% and the one for the blue environment to 90%. For more information see [Adjusting traffic flow with traffic dials](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-traffic-dial.html) in the documentation.
  
  


Gradually increase the traffic dial until 100% for the green Region.



![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/8f18e5dc-87c8-4229-b244-8ab4a3cbc65a)





Requests from a client who should be served by US-WEST-2 region:




```
$ for ((i=0;i<100;i++)); do curl http://aebd116200e8c28ad.awsglobalaccelerator.com/ --silent >> output.txt; done; cat output.txt | sort | uniq -c ; rm output.txt;
 100 requests processed in US-WEST-2 (GREEN Environment)
 ```
 
 
 Users that should be served from US-WEST-2 region all now have the green environment. Repeat the same process to update the application in EU-WEST-1 Region. If issues arise during the deployment, achieve rollback by updating the traffic dial to 0 for the green Region, adding the previous blue environment and change the traffic dial back to 100%.
 
 
 
 
 # Conclusion
 
 
 By following the write up above, you can quickly implement blue/green and canary deployments for multi-region applications using [AWS Global Accelerator](https://aws.amazon.com/global-accelerator/). This solution is easy to implement, and does not relay on DNS, so you are not impacted by DNS caching and long DNS TTLs. Beyond traffic dials and endpoint groups, Global Accelerator also provides many other features such as failover, client affinity, health checking, and DDoS protection.








 # IMPLEMENTATION OF MULTI REGION DEPLOYMENT OF AN ECOMMERCE WEBSITE ON AWS WITH GLOBAL ACCELERATOR

 ## Step 1: launch an intance in the us-east-1 region

 For this step, i used an already existing AMI on which my ecommerce website is installed to luanch an ec2 in the us-east-1a region


 ![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/ad6cd092-4c09-43fb-a32d-701292a759e8)




  ## Step 2: launch an rds intance(read replica) in the us-east-1 region 

  For this step, i also used an already existing rds snapshot which was connected to my AMI to luanch an rds instance in the us-east-1b region


  ![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/ffc52789-68d4-4325-be4b-a7b1a17743b8)





 To access my website, i simply paste the public ipv4 address of the instance in my browser



![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/1318247f-8416-42f4-aeb1-c1fa26cdcbcc)






## Step 3: launch an intance in the eu-central-1a region

For this step and step 4 below, i simply repeted the two first step above, but this time in eu-central-1 


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/25f8a905-274f-4de7-9aff-10d9a5937431)



## Step 4: launch an rds intance(read replica) in the eu-central-1b region

![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/e69366c4-ded3-4ee5-b586-7b6a7ec59d98)






## Step 5: Create and configure a Global Accelerator service

### This Global accelerator service connects the endpoints of my two ec2 server on which my ecommerce application is deployed in the two different regions.


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/0714755a-9f70-43ae-8500-539f10cfc9ca)


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/fe63ab41-d93f-4b27-9c99-13f68eabaa0a)


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/fc8c460a-1acf-435d-96fc-cc3fa2803e61)


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/6c5ad260-fb2d-4478-9dbd-e17f1fbb5915)


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/16179894-a66b-4a99-9944-2893b1ad2064)



![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/2eb9d460-e256-4d03-8c05-d65aac6a96ac)


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/31327665-2ab4-4434-aec9-4db19ce3f9bc)


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/65e32ea0-1136-47a2-bf9d-4637f428c2e7)



### Once you have created the Global accelerator service and the status shows "deployed", copy the the dns name and paste it in the browser to access the application.


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/4b9bdf12-8197-428c-a013-a029fd0e65a2)



![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/f6522787-22d8-42ab-a04f-03983dcc15a2)




### The Global Accelerator service is now directing traffic to both the us-east-1 and eu-central-1 regions.




## Step 6: Test the Global Accelerator for failover capability

By Failover, i mean, if for any reason, something happens to either one of the region, say eu-cenral-1, the service should be able to direct traffic to us-east-1 for me to still be able to access the application.

To cause this failover, i will delibrately, make the health target in eu-central-1 to be unhealthy, by editing the security group of the ec2 instance so that traffic is blocked. This action will prevent me from accessing the ecommerce application on the ipv4 address of the ec2 instance, when its pasted on the browser.


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/6d4c86c5-c961-43cd-b7f2-ece51ae3f3f0)



### To comfirm if the global accelerator service has redirected traffic to the us-east-1 region, paste the the dns of the global accerator in the browser.


![image](https://github.com/georgeonalo/multi-region-deployment-in-aws-with-aws-global-accelerator/assets/115881685/de086804-8168-4886-82e9-39d9f6d5fdcc)




Yessssssssssssssssssssssssssssssss! the service is working fine.






 

 
 
 

  
  
  












 

 
 
 

  
  
  
