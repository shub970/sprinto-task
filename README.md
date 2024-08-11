
## Problem Statement

For this exercise, imagine that you are working in a small team that is building a product like Sprinto. The goal is to think about one of the requirements of a product: policies (details hereafter). Specifically, we want you to think about the data model that ensures that we have all the required information. i.e. design the tables/attributes that you would use to store all the required information to achieve the required functionality. Apart from the data model, think about the services, API’s and other functions that you would write to achieve this functionality. You can ignore authorization and input-data-validation functions.

## Key Decisions

1.  Priorities
    1.  Transactional capabilities
	```
		1. Only one policy version should be active at any given time
		2. Ack requests for a new policy version on approval
		3. New policy addition to a staff role triggering Ack requests
	```

    2.  Availability
	```
		1. For auditors to view the info
		2. For the customer (the company) to view the compliance gaps
	```

2.  Requirements are on extending the functionality (increasing complex queries) while the data is mostly structured
   
**This indicates the use of a SQL database - ensures Consistency and Availability (CAP Theorem)**

3.  Callouts

			1. No context of the compliance framework is used (to reduce complexity)

## DB Schema

[Link](https://dbdiagram.io/d/Shubham_Laddha-66b7cfb38b4bb5230ec52593)

## Requirements

Here are the details of the business requirement:

1.  A company needs multiple types of policies to become SOC 2 compliant (Infosec policy, acceptable use policy, cryptographic policy, etc.)
	```
	1. Create policy_types for different policies under SOC 2
	```
  
2.  To make our customers’ lives easy, Sprinto comes built-in with a set of default policy templates - one for each type of policy. The concept of a policy template itself needn’t be visible to our customers and is useful only for us to generate these policies for them if they so choose.
	```
	1. Create a Policy_Template for each policy type 
 
	2. Can incrementally create more templates against a type with a version (only one is active at a time for a given type)

	3. Create Default_Configuration key values for a template_id
	```

3.  Sometimes, customers don’t want to use the policy that is generated from our templates and would like to use their own. Our design should be able to accommodate this requirement.
	```
	1. Create a new policy with template_id as NULL
	```
  
4.  When a new policy is created, it needs to be approved by a designated person (let's say the CTO of the company).
	```
	1.  A new policy is created with is_active = false and is_pending_approval = true and status = “pending_approval”
	    
	2.  template_id is chosen based on the user input and Policy_Configurations are defined
	    
	3.  From Employees, find the employee with designation = “CTO” and trigger an approval request with details - type_id, template_id (if available), re_ack_in_days, next_review_date

	4.  On approval, the current active policy is set is_active = false and the new policy is updated with is_active = true, is_pending_approval = false, policy_content from either the chosen template+configurations or custom wording
	```
	
5.  An approved policy needs to be acknowledged by all employees to the effect that they have read and agree to abide by the policies. Acknowledgement is a simple “I agree to abide by the policies” checkbox.
	```
	1. On policy approval, get all employees who have the role attached to this policy type

	2. For each of these employees, create Acknowledgement_Requests with trigger_type = “auto”, trigger_date = <current_date>, due_date = <curr_date>+30 days, is_acknowledged = false
	```
6.  Employees need to acknowledge policies at 2 different points of time. First, when a new employee joins, they need to acknowledge all the policies within 30 days of joining. Second, all employees need to acknowledge all policies periodically thereafter (at least once a year).
	```
	1. When a new Employee record gets created, fetch all the policies based on its role

	2. Create Acknowledgement_Requests for all the policies as described above

	3. For periodic acknowledgement, run a cron once every day at end of the day to check for Policy_Acknowledgements with ack_expiry_date <= current timestamp. Trigger Ack requests for these expired acknowledgements by creating an Acknowledgemnt_Request record.
    ```

7.  For audit reasons, we need to keep information of all the past acknowledgements of every employee.
    ```
	1.  Stored in the Policy_Acknowledgements table
    ```

8.  We need to know how our data model will support the ability to request acknowledgements from new employees when they join, and periodically thereafter (ex: annually). We also need to be able to track whether the acknowledgements are completed within 30 days from the time of request. We might have to raise the right kind of alerts if acknowledgements are not happening in time - like escalate this to a CXO in the company.
	```    
	1.  Both the cases, new joiners and existing employees re-ack process has been described above.
    
	2.  For alerts (eg to the CXO), run a cron every end of the day to get Acknowledgement_Requests whose due_date <= curr_date
    ```

9.  Sometimes, as compliance frameworks change, we’d like to update the policy templates. When we, as builders of Sprinto, change the template, we need to be able to show a prompt to our customers (the company) that they need to upgrade to the latest version of the policy wording. Before the upgrade happens the currently active policy in the company should continue to work.
	```  
	1.  Create a new Policy_Template for that specific policy type with is_active_version = false
    
	2.  Show prompt to the customer (in the dashboard) and wait for their consent
    
	3.  On consent, set is_active_version = false for the template which is currently active on that type and set is_active_version = true for this new template (transaction)
    
	4.  Now, any new policy gets created with this active template version of that policy type
    ```

10.  Some policies have specific configurations that are given by the customer at the time of creating the policy. For example, a vulnerability management policy will have SLAs for fixing vulnerabilities. This is an input that we need to receive from the customer when we generate the policy.
		````
		1.  Such inputs can directly be inserted into the Policy_configurations table against the newly created Policy. (The keys needed for a particular Policy template can be fetched from Default_Configurations table)
    
		2.  When this configuration is changed, the policy needs to be approved before the new policy takes effect. In the meantime, the previous policy should be functional i.e. if a new employee joins, they should be signing the version that is currently active.

		3. Policy_Configuration is versioned based on the policy_version. If a new configuration is to be created, a new config_id needs to be generated and the last policy_version is already in approved status. As policy_version has to be unique for a given policy_id, a policy_version needs to be created
    
		4.  Create a new policy version from the currently active version and add a new Policy_Configuration for the new version. The new policy version is in is_pending_approval = true state. So approval gets triggered
    
		5.  On approval, the current version is set is_active = false and the new version is set to is_active = true with status = “approved” (transaction)
		````

11.  Once the new policy is approved that should be the one used for any new hires joining thereafter.
		````
		1.  Already handled above as the active policy is picked against a given policy type
		````

12.  The same logic holds for periodic policy acknowledgements as well.
		````
		1.  Handled
		````
    

13.  When policy is modified, there is no hard and fast requirement for employees who acknowledged earlier to re-acknowledge. However, our data model should be able to handle this scenario if it comes up. Our customers should be able to request an acknowledgement only for the policies that were changed and not other policies.
		````
		1. If a policy is modified => a new policy version is created.
		2. Whether or not to trigger Acknowledge_Requests can be controlled by a flag.
		````
14.  We should be able to distinguish whether a policy acknowledgement is happening as a part of a periodic exercise or as part of a new joining. Apart from these 2 events, customers should be able to trigger an acknowledgement from selected employees manually. It is preferable if our data model can track this type of acknowledgement as well.
		````
		1.  Policy_Acknowlegements with created_at == updated_at, are the ones created for newly joined employees, rest are periodic ones
		    
		2.  To trigger a manual Ack, expose a function to create Acknowledgement_Requests for the selected employees based on the active version of the policy type.
		    
		3.  acknowledgement_type field tracks whether the ack was auto or manual
		````

15.  The set of policies to be acknowledged by different types of employees is different. For example, an HR employee may have to acknowledge only 3 policies while an engineering employee needs to acknowledge 15 policies. The list of policies to be acknowledged by different roles should be configurable.
	    ````
		 1. Handled in the Role_Policies table (Role_id to policy_id many to many mapping)
	    ````

  

## Services

####  Policy Type management
1.  Creation of new policy types
    `POST /policy/type`
    ```
    {
	    "name": "infosec"
	}
	```
2. Update policy type name
	`PUT /policy/type/:id`

#### Template management
    
1. Creation of Policy template version for each type
    `POST /policy/type/:policyId/templates`
	  ```
	  {
		  "templated_content": "This is a {{sample}} policy"
		  "defaultConfigs": {
			  "sample": "value"
			}
	  }  
	```
2. Activation of a created template
	`PUT /policy/templates/{id}/activate` 
	*sets the given template_id active and inactivates the current template_id for the same type_id*
    
3.  Define default configs for a template
	`PUT /templates/{id}/default-config`
		```
		{ 
			"sample" : "value1"
		}
		```
	*Can override existing keys*
    

#### Policy management
    

1.  Add a new policy
	`POST /company/:companyId/policy`
	```
	{
		"template_id?": "template:123",
		"type_id": "type:345",
		"content?": "Custom {{policy}} content",
		"configurations?": {
			"key1": "value1"
		},
		"acknowledgement_validation_period_days": 365
	}
	```
	*Also sends the CTO (queried from Employees) approval request*
    
2.  Approve a policy
	`PUT /policy/:policyId?action=approve`
	```
	{
		"next_review_in_days": 180
	}
	```
	*Validates if this policy is in 'pending_approval' state before approving*
	*On approval, policy_contents are updated based on either type or template_id*
    
3.  Modify a policy
	`PUT /policy/:policyId?action=alter`
	```
	{
		"template_id?": "template:123",
		"content?": "Custom {{policy}} content",
		"configurations?": {
			"key1": "value1"
		},
		"acknowledgement_validation_period_days?": 365
	}
	```
	*Modifying an existing policy creates a fork from that policy and return id of the new forked policy with the modifications*
  
  4.  Trigger Ackowledgement from employees
	`POST /company/:companyId/policy/:typeId/acknowledge`
		```
		{
			"employees": [
				"emp:123",
				"emp:345"
			]
		}
		```

#### Employee management
1. Add a new employee
	`POST /company/:companyId/employee`
	```
	{
		"name": "abcd",
		"role": "role:123",
		"designation": "CPO"
	}
	```
2. Attach policies to a role
	`PUT /roles/:roleId/policies`
	```
	{
		"policies": [
			"policytype:123",
			"policytype:786"
		}
	]
	```

## Functions
#### Enable Identification out-of-date acknowledgements
    This function shall schedule a DB event to execute every end of the day to query Policy_Acknowledgements with ack_expiry_at <= current_timestamp
    Trigger Acknowledgement request for such records

#### Auto Trigger alerts for expired Acknowledgement requests
	This shall schedule a DB event on every end of the day to re-trigger acknowledgement if its due_date >= current_date
