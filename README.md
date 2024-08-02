# Assignment 1: Database Queries

1.  Installed Docker on my Mac machine
2.  Installed MongoDB CLI with a package manager Homebrew

3.  Created a directory in my local using the command 
      ```sh 
      mkdir -p ~/jumpcloud-escalations-engineer-assignment/mongodb
      ```
4.  Changed directory to the workspace of my local Mongo DB 
      ```sh 
      cd /Users/keerthivignesh/jumpcloud-escalations-engineer-assignment/mongodb
      ```
5.  Pulled down mongo community image 
      ```sh
      docker pull mongo
      ```
6.  Replaced the path specified under volumes in the provided docker-compose.yml file to the path to the mongodb directory (/Users/keerthivignesh/jumpcloud-escalations-engineer-assignment/mongodb)
7.  Started the mongo service 
      ```sh
      docker-compose up -d
      ```
8. Got the container id for the new mongo container 
      ```sh 
      docker ps
      ``` 
   | CONTAINER ID|   IMAGE  |   COMMAND   |               CREATED  |       STATUS  |       PORTS          |            NAMES|
   |----|----|----|----|-----|-----|-----|
   |62b036326c88 |  mongo |    "docker-entrypoint.s…"  | 9 seconds ago |  Up 9 seconds |  0.0.0.0:27017->27017/tcp |  jcmongo|
9. Copied the provided .json files to the docker container by executing the below commands individually. 
      ```sh
      docker cp employees.json 62b036326c88:/
      ```
      ```sh
      docker cp beers.json 62b036326c88:/
      ```
      ```sh
      docker cp patrons.json 62b036326c88:/
      ```
10. Connected to the mongo docker container using 
      ```sh
      docker exec -it jcmongo bash
      ```
11. Imported the database backups by executing the commands individually. 
      ```sh
      mongoimport --db=brewery --collection=beers --file=beers.json
      ```
      ```sh
      mongoimport --db=brewery --collection=employees --file=employees.json
      ```
      ```sh
      mongoimport --db=brewery --collection=patrons --file=patrons.json
      ```
12. Checked the connection to the database 
      ```sh
      docker exec -it 62b036326c88 mongosh mongodb://localhost:27017/brewery
      ```
13. Successfully connected to the **Brewery** database from the terminal.
-----
&nbsp;
&nbsp;


# Assignment Questions:
1. We’re coming out with a new hoppy delicious IPA. To let our customers know, we need two mailing lists.

   a. One that includes the email addresses of all of our customers.

      - Command executed at the terminal
         ```sh
         db.patrons.find({}, { email: 1, _id: 0 })
         ```

   b. Another that includes only the email addresses for customers whose favorite beer is an IPA.
    ```sh
      db.patrons.aggregate([
      {
         $lookup: {
            from: "beers",
            localField: "favorite_beer",
            foreignField: "_id",
            as: "favorite_beer_details"
         }
      },
      {
         $match: {
            "favorite_beer_details.type": /IPA/i
         }
      },
      {
         $project: {
            email: 1,
            _id: 0
         }
      }
      ]).forEach(printjson)
      ```
-----
2. We need to gather some data on our tap rooms to show which is the most popular location. We need to know how many customers have frequented each location between 1/1/2024 - 4/1/2024.
   ```sh
   db.patrons.aggregate([
      {
         $match: {
            last_checkin: {
               $gte: new Date("2024-01-01"),
               $lt: new Date("2024-04-02")
            }
         }
      },
      {
         $group: {
            _id: "$location",
            totalVisits: { $sum: 1 }
         }
      },
      {
         $sort: { totalVisits: -1 }
      }
   ]).forEach(printjson)
   ```
-----

3. We’re trying to determine what type of beer is most popular with our customers so we can determine what our next experimental beer should be! Can you provide us with an array of objects that include the beer name, type, and number of customers where that beer is their favorite?
   ```sh
   db.patrons.aggregate([
      {
         $lookup: {
            from: "beers",
            localField: "favorite_beer",
            foreignField: "_id",
            as: "favoriteBeer"
         }
      },
      {
         $unwind: "$favoriteBeer"
      },
      {
         $group: {
            _id: {
               name: "$favoriteBeer.name",
               type: "$favoriteBeer.type"
            },
            customerCount: { $sum: 1 }
         }
      },
      {
         $project: {
            _id: 0,
            name: "$_id.name",
            type: "$_id.type",
            customerCount: 1
         }
      },
      {
         $sort: { customerCount: -1 }
      }
   ]).forEach(printjson)
   ```
-----
## Note: I used the below method for all the above queries to export the result to a .json file format.
1. Executed the command 
   ```sh
   docker exec -it 62b036326c88 bash
   ```
2. Created a JS file in the container's shell 
   ```sh
   cat << EOF > query.js
   db.patrons.find({}, { email: 1, _id: 0 }).forEach(printjson)
   EOF
   ```
3. From host machine run: 
   ```sh
   docker exec 62b036326c88 mongosh mongodb://localhost:27017/brewery query.js --quiet > email_list.json
   ```
   this is the query to export the file.
4. The **email_list.json** file contains the list of all e-mail address.

5. Created a JS file for IPA beer lovers 
   ```sh
   docker exec 62b036326c88 bash -c 'echo "db.patrons.aggregate([{\$lookup:{from:\"beers\",localField:\"favorite_beer\",foreignField:\"_id\",as:\"favorite_beer_details\"}},{\$match:{\"favorite_beer_details.type\":/IPA/i}},{\$project:{email:1,_id:0}}]).forEach(printjson)" > ipa_patrons_query.js'
   ```

6. Query to export the file 
   ```sh
   docker exec 62b036326c88 mongosh mongodb://localhost:27017/brewery ipa_patrons_query.js --quiet > ipa_patrons_emails.json
   ```
7. The **ipa_patrons_emails.json** file contains the list of e-mail addresses of customers whose favourite beer has the string IPA.

8. Created a JS file to know about customers who visited a location on a particular date 
   ```sh
   docker exec 62b036326c88 bash -c 'echo "db.patrons.aggregate([{\$match:{last_checkin:{\$gte:new Date(\"2024-01-01\"),\$lt:new Date(\"2024-04-02\")}}},{\$group:{_id:\"\$location\",totalVisits:{\$sum:1}}},{\$sort:{totalVisits:-1}}]).forEach(printjson)" > popular_locations_query.js'
   ```
9. Query to export the file 
   ```sh
   docker exec 62b036326c88 mongosh mongodb://localhost:27017/brewery popular_locations_query.js --quiet > popular_locations.json
   ```
10. The **popular_locations.json** file contains the list of customers who visited a location between 1/1/2024 - 4/1/2024.

11. Created a JS file for identifying the Most Popular Beer Among Customers 
      ```sh
      docker exec 62b036326c88 bash -c $'echo \'db.patrons.aggregate([{$lookup:{from:"beers",localField:"favorite_beer",foreignField:"_id",as:"favoriteBeer"}},{$unwind:"$favoriteBeer"},{$group:{_id:{name:"$favoriteBeer.name",type:"$favoriteBeer.type"},customerCount:{$sum:1}}},{$project:{_id:0,name:"$_id.name",type:"$_id.type",customerCount:1}},{$sort:{customerCount:-1}}]).forEach(printjson)\' > popular_beers_query.js'
      ```
12. To export the results 
      ```sh
      docker exec 62b036326c88 mongosh mongodb://localhost:27017/brewery popular_beers_query.js --quiet > popular_beers.json
      ```
13. The **popular_beers.json** is the file with the results.
------
# Assignment 2
## Using the programming language of your choice, complete the following and provide your solution for each: I have used REST API calls via Postman to execute each of the below:

1. In Postman, under variables I set my API key which I got from JumpCloud.
    ```
    -Headers:
    x-api-key: {{api_key}}
    Content-Type: application/json
    ```
---
2. Create and activate 2 Users.

- POST https://console.jumpcloud.com/api/systemusers

- Body:
    ```json
    {
    "username": "Keerthi_Vignesh",
    "email": "keerthichoolakkal@gmail.com",
    "firstname": "Keerthi",
    "lastname": "Vignesh",
    "password": "Password123!",
    "activated": true
    }
    ```

- I used similar POST commands to create 4 users in my Jumpcloud console. User activation is also done.
---

3. Create a Group of Users

- POST https://console.jumpcloud.com/api/v2/usergroups

- Body:
    ```json
    {
        "name": "Keerthi_Test_Group"
    }
    ```
- I created similar 2 user groups.

---

4. Associate Users to the Group of Users

- POST https://console.jumpcloud.com/api/v2/usergroups/66abae6f94ccbc0001533015/members

- Body:
    ```json
    {
      "op": "add",
      "type": "user",
      "id": "66abab173222f83277b296ee"
    }
    ```
- I similarly added 2 users to each groups.
---

5. Associate one User to the recently added system in JumpCloud
- I added my personal Mac device in the devices list and binded the users to the devices so that they can start using the devices.

- POST https://console.jumpcloud.com/api/v2/systems/66aba75ebae7dc6aa262e094/associations

- Body:
    ```json
    {
    "op": "add",
    "type": "user",
    "id": "66abd5062287da32aada43f3"
    }
    ```
- Similarly added the other users as well.
---

5. Log in to the system as the JumpCloud managed user- Successful.
---

6. Configure LDAP for your organization in the JumpCloud Admin Portal- Configured via Jumpcloud admin portal.
---

7. Add two user groups to your LDAP directory, each user group containing at least 2 unique users- Done.
---

8. Using a seperate BindDN user query your LDAP directory over SSL using 'ldapsearch' and show:

- All users bound to LDAP in your organization

    ```ldapsearch -H ldaps://ldap.jumpcloud.com:636 -x -b "ou=Users,o=66983c44bce0f42c5abcbcc5,dc=jumpcloud,dc=com" -D "uid=Keerthi_Vignesh,ou=Users,o=66983c44bce0f42c5abcbcc5,dc=jumpcloud,dc=com" -W "(objectClass=inetOrgPerson)"```

- Just the users in one specifc user group (your choice of the two groups you created) and only returning the users email, firstname, and lastname.
    ```ldapsearch -H ldaps://ldap.jumpcloud.com:636 \
    -D "uid=Keerthi_Vignesh,ou=Users,o=66983c44bce0f42c5abcbcc5,dc=jumpcloud,dc=com" \
    -w Password123! \
    -b "ou=Users,o=66983c44bce0f42c5abcbcc5,dc=jumpcloud,dc=com" \
    -s sub \
    "(&(objectClass=inetOrgPerson)(memberOf=cn=Test_Group,ou=Users,o=66983c44bce0f42c5abcbcc5,dc=jumpcloud,dc=com))" \
    mail givenName sn -LLL```
---

