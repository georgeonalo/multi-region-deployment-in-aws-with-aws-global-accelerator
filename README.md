# multi region deployment with aws global accelerator

To implement a multi region deployment, otherwise known as blue/green deployment across different AWS Regions, use [Global Accelerator traffic dials to dial](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-traffic-dial.html) up or down traffic to a specific AWS Region. For each AWS Region (or endpoint group), we set a traffic dial to control the percentage of traffic that is directed to that Region.
 
 
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

  
  
  
