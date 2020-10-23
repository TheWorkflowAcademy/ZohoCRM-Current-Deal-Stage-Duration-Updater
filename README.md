# ZohoCRM-Current-Deal-Stage-Duration-Updater
Deluge script that calculates the duration of Deals in their respective current Deal Stages.

## Core Idea
Zoho Reports has a limitation when it comes to getting the Deal Stage Duration - the system only captures stage duration when a Deal has exited a stage, which means, you can only get the duration of a Deal in its past stages. To get the **current** Deal stage duration, some Deluge scripting would be necessary. The idea is to get the timestamp of a Deal when it entered its **current** stage, calculate the duration between the aforementioned timestamp vs today (whenever the function is run), and update the value in a custom field on the Deal record.

## Configuration
Before you can use this script with Zoho CRM, you must configure the following:
* Create a Zoho CRM Connection with the following scope: **ZohoCRM.modules.ALL**.
* Create a custom line field in the Deals module called "Current Stage Duration (Days)".
* Create a schedule for which the custom function will be written in and run. It's advisable to have this schedule run on a daily basis (before work starts) for up-to-date information.
  * Settings > Automation > Schedules > Create New Schedule

## Tutorial
### Get all Deal Records
To get all Deal records, a Zoho CRM API call is used. Most companies have over 200 Deals in their CRM, but Zoho CRM API has a limit of 200. In order to overcome this, we would simulate a `while` loop via pagination. This allows you to get every single Deal record in a list (read more about API pagination here: https://github.com/TheWorkflowAcademy/api-pagination-zohocrm).

```javascript
zohoCrmModule = "Deals";
perPageLimit = 200;
pageIterationList = {1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25}; // This accounts for 5000 Deals (25 x 200). Increase the page number if needed.
allRecords = List();
iterationComplete = false;

for each  page in pageIterationList
{
	if(iterationComplete == false)
	{
		response = invokeurl
		[
			url :"https://www.zohoapis.com/crm/v2/" + zohoCrmModule + "?page=" + page + "&per_page=" + perPageLimit
			type :GET
			connection:"zohocrm" // Change this to your Connection Name
		];
		records = response.get("data");
		allRecords.addAll(records);
		if(records.size() < perPageLimit)
		{
			iterationComplete = true;
		}
	}
}
// Check for correctness
info "Number of Records: " + allRecords.size();
```

## Get the Current Deal Stage, Calculate the Duration, Update the Custom Field
* A `For` loop is used here to iterate through every single Deal record in the list. 
* To access the Deal Stage information, a Zoho CRM API call is used to get the Stage History (note: "Stage_History" is a related list in Deals).
* To get the **current** Deal Stage, you must get the **first index** (`.get(0)`) - by default, the latest (current) stage is the first index in the list. 
* Once the accurate Stage record is accessed, `.get("Last_Modified_Time")` will get the time stamp of when the Deal enters its current stage. 
* Calculate that time stamp vs `today` with the `days360` funciton and voila, you get the duration (in days) of the current Deal Stage.
* Update that value in the custom line field you have created.

```javascript
for each  a in allRecords
{
	//Get the current Deal Stage
	response2 = invokeurl
	[
		url :"https://www.zohoapis.com/crm/v2/Deals/" + a.get("id") + "/Stage_History"
		type :GET
		connection:"zohocrm"
	];
	//Calculate the Duration
	duration = days360(response2.get("data").get(0).get("Last_Modified_Time"),today);
	//Update field here
	update = zoho.crm.updateRecord("Deals",a.get("id"),{"Current_Stage_Duration_Days":duration}); // Change this to your custom field API name if needed.
	info update;
}
```

## Use the Custom Field in Zoho Reports
Now, you can use the "Current Stage Duration (Days)" custom field in Zoho Reports for analysis purposes. A good usage of this is for sales people to review dormant Deals.
