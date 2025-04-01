# SQL Murder Mystery

This is a fun game to practice your SQL and for strengtening your SQL concept and logic. The game is called "SQL Murder Mystery" and can be played from this website [https://mystery.knightlab.com/](https://mystery.knightlab.com/).

## Premise :

A crime has taken place and the detective needs your help. The detective gave you the crime scene report, but you somehow lost it. You vaguely remember that the crime was a ​murder​ that occurred sometime on ​Jan.15, 2018​ and that it took place in ​SQL City​. Start by retrieving the corresponding crime scene report from the police department’s database.

## Database Structure

You can play by looking or not looking at this diagram. Both are completely fine.
![database](image/schema.png)

## Solution Steps

### Initial Crime Scene Report

```sql
SELECT * FROM crime_scene_report WHERE city = "SQL City" AND DATE = '20180115' AND type = 'murder';
```

Output :

Security footage shows that there were 2 witnesses.
The first witness lives at the last house on "Northwestern
Dr". The second witness, named Annabel, lives somewhere on
"Franklin Ave".

### Finding The first witness's information

The crime report shows that the first witness lives at the last house on "Northwestern Dr". We can get the name of the individual by searching by the address street name column and fetch the row with the highest address number.

```sql
SELECT * FROM person WHERE address_street_name = "Northwestern Dr" ORDER BY address_number DESC LIMIT 1;
```

| id    | name           | license_id | address_number | address_street_name | ssn       |
| ----- | -------------- | ---------- | -------------- | ------------------- | --------- |
| 14887 | Morty Schapiro | 118009     | 4919           | Northwestern Dr     | 111564949 |

### Finding the second witness's information

The crime report shows that the second witness is named "Annabel" and lives in "Franklin Ave". We can get her information by searching the name using wildcards and filter it by the street name.

```sql
SELECT * FROM person WHERE address_street_name = "Franklin Ave" AND name LIKE "%Annabel%";
```

| id    | name           | license_id | address_number | address_street_name | ssn       |
| ----- | -------------- | ---------- | -------------- | ------------------- | --------- |
| 16371 | Annabel Miller | 490173     | 103            | Franklin Ave        | 318771143 |

### Finding the interview transcript from both witnesses.

We can use many methods, but in this example i use CTE to find the id matching the previous query in the interview table.

```sql
WITH Morty_Query AS (
  SELECT id FROM person WHERE address_street_name = "Northwestern Dr" ORDER BY address_number DESC LIMIT 1
)

SELECT * FROM interview WHERE person_id = (
  SELECT id FROM Morty_Query
);
```

| person_id | transcript                                                                                                                                                                                                                      |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 14887     | I heard a gunshot and then saw a man run out. He had a "Get Fit Now Gym" bag. The membership number on the bag started with "48Z". Only gold members have those bags. The man got into a car with a plate that included "H42W". |

```sql
WITH Annabel_Query AS (
  SELECT * FROM person WHERE address_street_name = "Franklin Ave" AND name LIKE "%Annabel%"
)

SELECT * FROM interview WHERE person_id = (
  SELECT id FROM Annabel_Query
);
```

| person_id | transcript                                                                                                            |
| --------- | --------------------------------------------------------------------------------------------------------------------- |
| 16371     | I saw the murder happen, and I recognized the killer from my gym when I was working out last week on January the 9th. |

### Finding the person from Morty's transcript.

The person in his testimony is described as person who has an exclusive bag that can only belonged to a member that have "Gold" membership. He also describe other clues such as that the membership number on the bag starts with "48Z" and he got into a car that has "H42W" on its license plate.

We can proceed with the investigation with those clues. We can get the information by filtering the membership table by the 'gold' type and then find the id that starts with "48Z".

```sql
SELECT * FROM get_fit_now_member WHERE membership_status = 'gold' AND id LIKE "48Z%";
```

| id    | person_id | name          | membership_start_date | membership_status |
| ----- | --------- | ------------- | --------------------- | ----------------- |
| 48Z7A | 28819     | Joe Germuska  | 20160305              | gold              |
| 48Z55 | 67318     | Jeremy Bowers | 20160101              | gold              |

But if we try to find the drivers license information by joining both table it shows no result because there's no information to connect these two tables as the id in drivers license table is using an ID for identifying those individual licenses, not the person owning it.

```sql
SELECT * FROM get_fit_now_member gfm JOIN drivers_license dl ON gfm.person_id = dl.id WHERE gfm.membership_status = 'gold' AND dl.id LIKE "48Z%";
```

No data returned

So we can try to get the information on the driver license table by only filtering the license that has "H42W" in their license plate.

```sql
SELECT * FROM drivers_license WHERE plate_number LIKE "%H42W%";
```

| id     | age | height | eye_color | hair_color | gender | plate_number | car_make  | car_model |
| ------ | --- | ------ | --------- | ---------- | ------ | ------------ | --------- | --------- |
| 183779 | 21  | 65     | blue      | blonde     | female | H42W0X       | Toyota    | Prius     |
| 423327 | 30  | 70     | brown     | brown      | male   | 0H42W2       | Chevrolet | Spark LS  |
| 664760 | 21  | 71     | black     | black      | male   | 4H42WR       | Nissan    | Altima    |

### Finding the person from Annabel's transcript.

In her testimony, she said that she recognized the murderer because she saw the person in the gym that she's working out from on the January 9th. We can get the information by using substr() in the check-in date column in the check in table. Substr() is used because the website use sqlite. The format of the date is "YYYYDDMM" so we can filter the information by taking the substring of the last 4 char.

```sql
SELECT * FROM get_fit_now_check_in WHERE substr(check_in_date, -4) = "0901"
```

| membership_id | check_in_date | check_in_time | check_out_time |
| ------------- | ------------- | ------------- | -------------- |
| VHN8M         | 20170901      | 513           | 1013           |
| C581L         | 20170901      | 913           | 954            |

We can get the person's information by joining it with the membership table.

```sql
SELECT * FROM get_fit_now_member gfm
LEFT JOIN get_fit_now_check_in gfc ON gfm.id = gfc.membership_id
WHERE gfm.membership_status = 'gold' AND gfm.id LIKE "48Z%" AND substr(gfc.check_in_date, -4) = '0109'
```

| id    | person_id | name          | membership_start_date | membership_status | membership_id | check_in_date | check_in_time | check_out_time |
| ----- | --------- | ------------- | --------------------- | ----------------- | ------------- | ------------- | ------------- | -------------- |
| 48Z7A | 28819     | Joe Germuska  | 20160305              | gold              | 48Z7A         | 20180109      | 1600          | 1730           |
| 48Z55 | 67318     | Jeremy Bowers | 20160101              | gold              | 48Z55         | 20180109      | 1530          | 1700           |

We can match the output of this query and find further information by joining it with the Person table.

```sql
WITH gfm_gfc_cte AS (
  SELECT * FROM get_fit_now_member gfm
  LEFT JOIN get_fit_now_check_in gfc ON gfm.id = gfc.membership_id
  WHERE gfm.membership_status = 'gold' AND gfm.id LIKE "48Z%" AND substr(gfc.check_in_date, -4) = '0109'
)


SELECT p.id, p.name, p.license_id, p.address_number, p.address_street_name, p.ssn, cte.membership_id FROM person p JOIN gfm_gfc_cte cte ON p.id = cte.person_id;
```

| id    | name          | license_id | address_number | address_street_name   | ssn       | membership_id |
| ----- | ------------- | ---------- | -------------- | --------------------- | --------- | ------------- |
| 28819 | Joe Germuska  | 173289     | 111            | Fisk Rd               | 138909730 | 48Z7A         |
| 67318 | Jeremy Bowers | 423327     | 530            | Washington Pl, Apt 3A | 871539279 | 48Z55         |

### Narrowing down the suspect

Finally, we can get the vehicle / license information by joining it with driver license table.

```sql
WITH gfm_gfc_cte AS (
  SELECT * FROM get_fit_now_member gfm
  LEFT JOIN get_fit_now_check_in gfc ON gfm.id = gfc.membership_id
  WHERE gfm.membership_status = 'gold' AND gfm.id LIKE "48Z%" AND substr(gfc.check_in_date, -4) = '0109'
), person_member AS (
        SELECT p.id, p.name, p.license_id, p.address_number, p.address_street_name, p.ssn, cte.membership_id FROM person p JOIN gfm_gfc_cte cte ON p.id = cte.person_id
)

SELECT * FROM drivers_license dl JOIN person_member pm ON dl.id = pm.license_id;
```

| id     | age | height | eye_color | hair_color | gender | plate_number | car_make  | car_model | id    | name          | license_id | address_number | address_street_name   | ssn       | membership_id |
| ------ | --- | ------ | --------- | ---------- | ------ | ------------ | --------- | --------- | ----- | ------------- | ---------- | -------------- | --------------------- | --------- | ------------- |
| 423327 | 30  | 70     | brown     | brown      | male   | 0H42W2       | Chevrolet | Spark LS  | 67318 | Jeremy Bowers | 423327     | 530            | Washington Pl, Apt 3A | 871539279 | 48Z55         |

We found the murderer. His name is "Jeremy Bowers".

But it's not over yet. If we post the answer, we can see that the murderer is not the brain behind the crime.

```sql
INSERT INTO solution VALUES (1, 'Jeremy Bowers');
SELECT value FROM solution;
```

| value                                                                                                                                                                                                                                                                                                                                                                                              |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Congrats, you found the murderer! But wait, there's more... If you think you're up for a challenge, try querying the interview transcript of the murderer to find the real villain behind this crime. If you feel especially confident in your SQL skills, try to complete this final step with no more than 2 queries. Use this same INSERT statement with your new suspect to check your answer. |

We can get further information by looking into his interview transcription. We can do that by using the id by joining the previous query with the interview table.

### The Murderer's Transcript

```sql
WITH gfm_gfc_cte AS (
  SELECT * FROM get_fit_now_member gfm
  LEFT JOIN get_fit_now_check_in gfc ON gfm.id = gfc.membership_id
  WHERE gfm.membership_status = 'gold' AND gfm.id LIKE "48Z%" AND substr(gfc.check_in_date, -4) = '0109'
), person_member AS (
        SELECT p.id, p.name, p.license_id, p.address_number, p.address_street_name, p.ssn, cte.membership_id FROM person p JOIN gfm_gfc_cte cte ON p.id = cte.person_id
), murderer_id AS(
  SELECT pm.id FROM drivers_license dl JOIN person_member pm ON dl.id = pm.license_id
)

SELECT i.person_id, i.transcript FROM interview i JOIN murderer_id m ON i.person_id = m.id;
```

| person_id | transcript                                                                                                                                                                                                                                       |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 67318     | I was hired by a woman with a lot of money. I don't know her name but I know she's around 5'5" (65") or 5'7" (67"). She has red hair and she drives a Tesla Model S. I know that she attended the SQL Symphony Concert 3 times in December 2017. |

There's plenty of clues that the murderer give. That is the brain behind this crime has:

- Height: 65" or 67" or something in between.
- Hair Color : Red
- Car : Tesla Model S
- Attended the SQL Symphony Concert 3 times in December 2017

### Finding the person described from the murderer's description

We can try to find the information in the driver license table as the table has 3 of those as a column.

```sql
SELECT *
FROM drivers_license
WHERE height BETWEEN 65 AND 67
  AND hair_color = 'red'
  AND car_make = 'Tesla'
  AND car_model = 'Model S';
```

| id     | age | height | eye_color | hair_color | gender | plate_number | car_make | car_model |
| ------ | --- | ------ | --------- | ---------- | ------ | ------------ | -------- | --------- |
| 202298 | 68  | 66     | green     | red        | female | 500123       | Tesla    | Model S   |
| 291182 | 65  | 66     | blue      | red        | female | 08CM64       | Tesla    | Model S   |
| 918773 | 48  | 65     | black     | red        | female | 917UU3       | Tesla    | Model S   |

### Finding the person who attended the orchestra 3 times

We've narrowed down to 3 individual. So we will narrow down the list of suspects by filtering them by the number of times they visited the SQL Symphony Concert in December 2017.

```sql
SELECT person_id, COUNT(person_id) AS attendance_count FROM facebook_event_checkin
WHERE event_name = "SQL Symphony Concert" AND substr(date, 1, 6) = "201712"
GROUP BY person_id
HAVING attendance_count >= 3
ORDER BY attendance_count DESC
```

| person_id | attendance_count |
| --------- | ---------------- |
| 24556     | 3                |
| 99716     | 3                |

There's only 2 person that attended the concert 3 times.

### Narrowing down the 2 suspects

We can first find the information by joining the table with person table by matching the id from previous query.

```sql
WITH sql_att_count AS(
        SELECT person_id, COUNT(person_id) AS attendance_count FROM facebook_event_checkin
        WHERE event_name = "SQL Symphony Concert" AND substr(date, 1, 6) = "201712"
        GROUP BY person_id
        HAVING attendance_count >= 3
        ORDER BY attendance_count DESC
)

SELECT * FROM person p JOIN sql_att_count sac ON p.id = sac.person_id
```

| id    | name             | license_id | address_number | address_street_name | ssn       | person_id | attendance_count |
| ----- | ---------------- | ---------- | -------------- | ------------------- | --------- | --------- | ---------------- |
| 24556 | Bryan Pardo      | 101191     | 703            | Machine Ln          | 816663882 | 24556     | 3                |
| 99716 | Miranda Priestly | 202298     | 1883           | Golden Ave          | 987756388 | 99716     | 3                |

In this list only one woman appears, which match the murderer's transcript. But just to be safe, let's join this query with the driver's license query that has been filtered by the traits described by the murderer (Eventhough joining it using only the id suffice).

```sql
WITH sql_att_count AS(
        SELECT person_id, COUNT(person_id) AS attendance_count FROM facebook_event_checkin
        WHERE event_name = "SQL Symphony Concert" AND substr(date, 1, 6) = "201712"
        GROUP BY person_id
        HAVING attendance_count >= 3
        ORDER BY attendance_count DESC
), sql_person_count AS(
  SELECT * FROM person p JOIN sql_att_count sac ON p.id = sac.person_id
)


SELECT * FROM drivers_license dl JOIN sql_person_count spc ON dl.id = spc.license_id WHERE dl.height BETWEEN 65 AND 67
  AND dl.hair_color = 'red'
  AND dl.car_make = 'Tesla'
  AND dl.car_model = 'Model S';
```

| id     | age | height | eye_color | hair_color | gender | plate_number | car_make | car_model | id    | name             | license_id | address_number | address_street_name | ssn       | person_id | attendance_count |
| ------ | --- | ------ | --------- | ---------- | ------ | ------------ | -------- | --------- | ----- | ---------------- | ---------- | -------------- | ------------------- | --------- | --------- | ---------------- |
| 202298 | 68  | 66     | green     | red        | female | 500123       | Tesla    | Model S   | 99716 | Miranda Priestly | 202298     | 1883           | Golden Ave          | 987756388 | 99716     | 3                |

We've found the name of the brain behind the murder. Her name is "Miranda Prisetly".

### Submitting The answer

```sql
INSERT INTO solution VALUES (1, 'Miranda Priestly');
SELECT value FROM solution;
```

| value                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Congrats, you found the brains behind the murder! Everyone in SQL City hails you as the greatest SQL detective of all time. Time to break out the champagne! |

That concludes this simple web based SQL game. Note that this readme is made for fun and i'm using this to test / learn the application of SQL further.

## _Thanks For Reading_
