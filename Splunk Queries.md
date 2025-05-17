- **US Bank Package Failed Creation**  
  Identifies failed disclosure package creations for US Bank loans.

  index=blend-sensitive-prod-blend-bailey-web parsed.task.processId="Disclosures*" parsed.tenant=usbank parsed.task.loanRequest.requirement.disclosuresPackageId=* 
| rename parsed.task.loanRequest.requirement.disclosuresPackageId as packageID | search packageID=*
    [ search index=blend-sensitive-prod "kubernetes.container_name"=disclosures "parsed.tenant"=usbank "parsed.message"="Status for package * is FAILED_TO_CREATE" 
    | rex "Status for package (?P<packageID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12}) is FAILED_TO_CREATE" 
    | table packageID]
    | dedup parsed.loanId
    | table _time parsed.tenant parsed.loanId parsed.envelopeId parsed.task.requestId
    | sort - _time

- **Citizens Alert: Email Template Config Change**  
  Tracks configuration changes to Citizens One email templates.

  index="blend-sensitive-prod-events" "body.name"=EmailTemplateConfig "body.tenants{}"=citizensone | rename body.tenants{} as Client, body.updatedAt as Timestamp, body.action as Action, body.name as Config, meta.env as Env | table Timestamp Action Config Env Client

- **Citizens HE Pricing**  
  Flags ESOCKETTIMEDOUT errors from Citizens HE pricing engine.

  index="blend-sensitive-prod-blend-bailey-web" citizensbank-he "Pricing Engine is Unavailable" "Error: ESOCKETTIMEDOUT" | dedup parsed.err.params.loanId | table _time parsed.err.params.loanId parsed.err.params.errorClientName parsed.err.code

- **Citizens HE 412 PATCH**  
  Detects 412 PATCH errors during Empower integration for Citizens HE.

  index="blend-sensitive-prod" 412 PATCH citizensbank-he "parsed.level"=error "Error running empower integration for citizensbank-he on Blend loan" 
| rex "Error running empower integration for citizensbank-he on Blend loan (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rex ": Empower: (?P<error>Unexpected error code 412 for PATCH https...api.bkiconnect.com.empowerWebapi.v1.CitizensHE.WebApi.BlendRetail.Loans...............Borrowers..\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12}..)"
| table _time loanID error

- **Citizens HE Exports**  
  Finds failed Fannie export attempts for Citizens HE.

  index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: FannieExport*" citizensbank-he
| rex "Export for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time loanID parsed.message

- **Citizens Mortgage Exports**  
  Finds failed MISMO exports for Citizens One mortgages.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" citizensone
| rex "Export for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time loanID parsed.message

- **53 Exports**  
  Tracks failed MISMO exports for Fifth Third Bank.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" fifththird
| rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time parsed.message loanID

- **Gateway Export Failures**  
  Identifies Gateway Empower MISMO export failures.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" gatewayempower
| rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time parsed.message loanID

- **Huntington Export Failures**  
  Detects Mulesoft export errors for Huntington loans.

index="blend-sensitive-mulesoft-prod" "Mulesoft error payload for loan-export" message.statusCode=500 huntington
| rex "trackingId....(?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rex "\S\W{5}\S{5}\W\W{3}\W\w{3}\W(?P<los>[a-z]{3,})"
| rex "\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12}__(?P<eventID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rename message.tenant as tenant | dedup loanID | table _time loanID eventID los tenant | sort - _time

- **JB Nutter Export Failures**  
  Tracks failed MISMO exports for JB Nutter.

index="blend-sensitive-prod" jbnutter parsed.level=error "EmpIntg: failed performing action: MISMOExport" | rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" | dedup loanID | table _time loanID parsed.deployment

- **Keybank ESOCKETTIMEDOUT**  
  Flags KeyBank export timeouts due to ESOCKETTIMEDOUT errors.

index="blend-sensitive-prod" ESOCKETTIMEDOUT parsed.tenant=key parsed.details.event.data.eventType="applicationExportRequested" "parsed.dd.service"="beehive-retryworkers" 
| rex "los-integrations.blendlabs.com:443...(?P<error>\w{5}..\w{15})" 
| dedup parsed.details.event.data.data.loanId 
| rename parsed.details.event.data.data.loanId as loanId, parsed.details.event.data.data.exportRequestReason as exportReason, parsed.details.event.data.eventType as eventType, parsed.tenant as client
| table _time client loanId error exportReason eventType

- **MTB Export Failures**  
  Identifies failed MISMO export events for MTB.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" mtb
| rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time loanID parsed.message

- **Mr Cooper mrcCustomerId Failed**  
  Joins and displays failed customer ID lookups for Mr Cooper.

index="blend-sensitive-prod-blend-bailey-web" parsed.loanId=* parsed.borrowerId=*
parsed.tenant=mrcooper kubernetes.container_name=blend-bailey-web
| rename parsed.borrowerId as partyId | join partyId
[search index="blend-sensitive-prod-blend-bailey-web" error mrcCustomerId parsed.tenant=mrcooper "/api/external/parties/*"
| rex "/api/external/parties/(?P<partyId>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" | dedup partyId | table partyId parsed.err.message]
| dedup parsed.loanId
| table _time parsed.tenant parsed.err.message parsed.loanId partyId

- **Navy Fed Export Failures**  
  Flags failed export events for Navy Federal.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" navyfederal
| rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time loanID parsed.message

- **PennyMac Completed Export Durations**  
  Calculates export duration time for PennyMac exports.

index=blend-sensitive-prod parsed.loanId=*
"Processing webhook" parsed.eventType="applicationExport*" parsed.tenant=pennymac-new
| transaction parsed.loanId startswith="parsed.eventType=applicationExport" endswith="parsed.eventType=applicationExported" 
| dedup parsed.loanId sortby +parsed._timestamp | table parsed.loanId parsed._timestamp duration | eval Export_Duration_Minutes=(duration)/60 | sort - duration
| rename parsed._timestamp as Start/End, duration as Export_Duration_Seconds, parsed.loanId as Loan_ID

- **Regions Export Failures**  
  Finds failed MISMO exports for Regions Bank.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" regions
| rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time loanID parsed.message

- **Mulesoft Monitoring Export Failures**  
  Monitors Mulesoft payload errors across LOS integrations.

index=blend-sensitive-mulesoft-prod source="http:firehose-mulesoft-text-logs" 
message.text="Mulesoft error payload for loan-export" message.statusCode=500
earliest="12/5/2021:01:00:00" latest="12/9/2021:23:00:00"
| rex "trackingId....(?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rex "\S\W{5}\S{5}\W\W{3}\W\w{3}\W(?P<los>[a-z]{3,})"
| rex "\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12}__(?P<eventID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rename message.tenant as tenant | dedup loanID | table _time loanID eventID los tenant | sort - _time

- **Synovus Export Failures**  
  Flags failed MISMO exports for Synovus.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" synovus
| rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time parsed.message loanID

- **Truist Assignment Failures**  
  Detects invalid NMLS assignment attempts for Truist.

index="blend-sensitive-prod" "EmpIntg: failed performing action: NMLS for *. error: Invalid request" parsed.deployment=suntrust | rex "EmpIntg: failed performing action: NMLS for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" | dedup parsed.message | table _time parsed.message loanID

- **Truist Borrower Sync Failures**  
  Tracks borrower sync failures for Truist with Wolters Kluwer.

index="blend-sensitive-prod" parsed.tenant=suntrust "/api/wolters-kluwer" 500 "Error: Unable to sync borrower" NOT "Duplicate or incomplete position data provided"
 | rex "borrower-sync.k8s.prod.blend.com.api.loans.(?P<loanId>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" 
 | rex "borrowers/sync: 4.. -....error.....(?P<error>.*?)\"\,"
 | dedup loanId
 | table _time loanId error

- **Truist Doc Sync Failures**  
  Finds document sync failures for Truist.

 index="blend-sensitive-prod" "EmpIntg: failed to sync doc" suntrust NOT Reattempt
| rex "sync doc for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rename parsed.err{}.message as error
| table _time loanID error
| sort - _time

- **Truist Doc Screenshot**  
  Monitors document screenshots taken from Blend Mobile for Truist.

index="blend-sensitive-prod-events" body.verb=DOCUMENT_SCREENSHOT_TAKEN body.tenant=suntrust body.object.clientName="Blend Mobile" body.object.documentId=* body.object.fileName=* body.object.loanId=* body.object.userIP=* body.publishedAt=* meta.env=prod
| rename body.verb as Action, body.tenant as Client, body.object.clientName as App, body.object.documentId as DocId, body.object.fileName as FileName, body.object.loanId as LoanId, body.object.userIP as UserIP, body.publishedAt as TimeStamp, body.actor as UserID
| dedup DocId | table TimeStamp, LoanId, DocId, UserID, Action, App, FileName, Client

- **Truist Export Failures**  
  Flags failed MISMO exports for Truist.

index="blend-sensitive-prod" parsed.message="EmpIntg: failed performing action: MISMOExport*" suntrust
| rex "MISMOExport for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID | table _time loanID parsed.message

- **Truist Licensing Checks**  
  Identifies licensing check failures for Truist.

index="blend-sensitive-prod" "LicensingCheckFailed" parsed.deployment=suntrust 
| rex "for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" 
| rex "message.........(?P<Error>\w{20})" 
| dedup loanID | table _time loanID Error

- **Truist Pricing Failures**  
  Tracks failed ApplyPricing actions for Truist.

index="blend-sensitive-prod" suntrust "EmpIntg: failed performing action: ApplyPricing"
| rex "ApplyPricing for (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| dedup loanID
| table _time loanID parsed.message

- **Truist WK Failures**  
  Aggregates Wolters Kluwer sync issues for Truist.

index="blend-sensitive-prod" parsed.tenant=suntrust "parsed.originalUrl"="/api/wolters-kluwer" "parsed.dd.service"="docgen-integrators" error NOT ("Multiple package mappings match" OR "recipient") | stats count by parsed.err.message parsed.message

- **US Bank Doc Screenshot**  
  Monitors US Bank document screenshots from Blend Mobile.

index="blend-sensitive-prod-events" body.verb=DOCUMENT_SCREENSHOT_TAKEN body.tenant=usbank body.object.clientName="Blend Mobile" body.object.documentId=* body.object.fileName=* body.object.loanId=* body.object.userIP=* body.publishedAt=* meta.env=prod
| rename body.verb as Action, body.tenant as Client, body.object.clientName as App, body.object.documentId as DocId, body.object.fileName as FileName, body.object.loanId as LoanId, body.object.userIP as UserIP, body.publishedAt as TimeStamp, body.actor as UserID
| dedup DocId | table TimeStamp, LoanId, DocId, UserID, Action, App, FileName, Client

- **US Bank Failued Package Creation**  
  Tracks US Bank packages that failed to create.

index="blend-sensitive-prod-events" body.context.tenant=usbank body.status=FAILED_TO_CREATE body.context.eventType=packageStatusUpdate | dedup body.applicationId
| rename body.context.tenant as client, body.packageId as packageId, body.applicationId as applicationId, body.context.eventType as eventType, body.context.eventProducer as eventProducer, body.loanType as loanType, body.status as status, body.packageType as packageType
| table _time client packageId applicationId eventType eventProducer loanType status | sort - _time

- **US Bank Mobile Screenshot**  
  Finds screenshots tied to US Bank disclosures and files.

index=blend-sensitive-prod (kubernetes.container_name=disclosures OR kubernetes.container_name=files) parsed.document.primaryFile.url=* 
    [ search index="blend-sensitive-prod" "usbank/documents/" parsed.apiRequestId=* 
        [ search index="blend-sensitive-prod" parsed.deployment=usbank parsed.originalUrl="/v1/internal/api/activities/document-screenshot" 
        | dedup parsed.apiRequestId 
        | table parsed.apiRequestId] 
    | rename parsed.keys{} as parsed.document.primaryFile.url 
    | dedup parsed.document.primaryFile.url 
    | table parsed.document.primaryFile.url] 
| rename parsed.document.displayFileName as fileType, parsed.document.fileName as fileName, parsed.document.primaryFile.createdBy as userId, parsed.loanId as loanId, parsed.tenant as tenant, parsed.document._id as docId
| dedup docId | table _time loanId docId fileName fileType userId tenant | sort - _time

- **US Bank Connection Failures**  
  Counts US Bank export errors caused by network timeouts.

index="blend-sensitive-prod" usbank "Unable to export event" ("ECONNREFUSED" OR "ESOCKETTIMEDOUT") | stats count by parsed.err.message

- **WF CRV Integration Failures**
  Helps identify failures with WF CRV prefill.

index=blend-sensitive-prod parsed.level=error kubernetes.container_name="wells-consumer-banking" 
parsed.statusText="Bad Request" parsed.message="validation error"
[search index=blend-sensitive-prod POST "/wells-fargo-personal-loan/applications" "914191e433594e01bbff397189a7ded1" 
"wells-consumer-banking" "mTLS validation failed but tenant has permissive configuration" 
| dedup parsed.dd.trace_id | table parsed.dd.trace_id] | rename parsed.errors{}.field{} as err_flds, 
parsed.errors{}.location as err_location, parsed.errors{}.messages{} as err_info, parsed.statusText as err_status, 
parsed.status as err_code, parsed.message as err_msg, parsed.name as err_name
| table _time err_flds err_location err_info err_status err_code err_msg err_name

- **WF Connectivity 500 Errors**
  Help identify 500 errors with WF CX.

index="blend-sensitive-prod" blend-wellsfargo-web ("personal-loan.wf.com" OR "yourmortgageapp.wf.com") [search index="blend-sensitive-prod" wellsfargo "Connectivity service error (500)" | table parsed.dd.trace_id] | dedup parsed.req.params.id | table _time parsed.req.params.id parsed.req.host

- **WF Beta RISE Errors**
  Simple query that identifies error being returned from WF RISE.

index=blend-internal-beta RISE "RISE Technical Error" "kubernetes.container_name"=wells parsed.tenant=* | table parsed.message parsed.tenant _time

- **WF Export Duration Threshold**
  This query measure the time that exports are taking from submission to export completed.

index=blend-sensitive-mulesoft-prod "blend-api-post-status-updates" service="blend-consumer-banking-prc-api" message.tenant="wellsfargo" message.route="/api/wellsCB/events"
| rex "trackingId....(?P<loanId>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| eval started=if(searchmatch("Update BONS status to PROCESSING"), 1, 0), succeeded=if(searchmatch("Update BONS status to SUCCEEDED"), 1, 0)
| stats earliest(_time) as export_start_time, latest(_time) as export_complete_time, sum(started) as export_triggered, sum(succeeded) as export_succeeded, range(_time) as export_duration by loanId
| fieldformat export_start_time=strftime('export_start_time',"%c")
| fieldformat export_complete_time=strftime('export_complete_time',"%c")
| where export_triggered=1 AND export_succeeded=1 AND export_duration>300

- **WF Soft Credit Pulls**
  Identifies the number of soft credit that have occurred for a timeframe.

index=blend-sensitive-prod wellsfargopilot "parsed.message"="Reporting credit action request for loan" "parsed.pullType"=Soft | table _time parsed.loanId parsed.creditActionType parsed.pullType parsed.creditPullMode parsed.provider parsed.tenant parsed.status | sort - _time

- **WF FNMA GSE Traffic**
  Identifies the number of FNMA XML files that were received over specified timeframe.

index=blend-sensitive-prod FNMA_NETPAY_GSE_API_XML wellsfargo "Webhook of type applicationFileAvailable received" | rex "loanId.....(?P<loan>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"| table _time parsed.message parsed.tenantId loan

- **WF Export Delays**
  Triggers an alert if the export completed event is taking longer than the average.

index=blend-sensitive-mulesoft-prod "blend-api-post-status-updates" service="blend-consumer-banking-prc-api" message.tenant="wellsfargo" message.route="/api/wellsCB/events"
| rex "trackingId....(?P<loanId>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| eval started=if(searchmatch("Update BONS status to PROCESSING"), 1, 0), succeeded=if(searchmatch("Update BONS status to SUCCEEDED"), 1, 0)
| stats earliest(_time) as export_start_time, latest(_time) as export_complete_time, sum(started) as export_triggered, sum(succeeded) as export_succeeded, range(_time) as export_duration by loanId
| fieldformat export_start_time=strftime('export_start_time',"%c")
| fieldformat export_complete_time=strftime('export_complete_time',"%c")
| where export_triggered=1 AND export_succeeded=1 AND export_duration>300

- **WF Package Traffic**
  Simple query that identifies completed closing packages.

index=blend-sensitive-prod parsed.data.packageType=CLOSING_PACKAGE parsed.tenant=wellsfargo parsed.data.status=COMPLETED kubernetes.container_name=disclosures parsed.data.loanType=PERSONAL_LOAN parsed.originalUrl="/api/events/envelopes" | stats count by parsed.data.packageType

- **WF Rate Checks**
  Simple query that identifies the number of successful rate check responses.

index="blend-sensitive-prod" "Successfully ingested rate check response with status: SUCCESS for loanId" "kubernetes.container_name"="wells-consumer-banking" "parsed.tenant"=wellsfargo

- **WF Signed/Viewed Packages Not Completed**
  This query measures the time between the signed/viewed events and the completed package event, if the completed package event does not trigger within an hour of the final signed event, there is an issue and this query helps identify this when it happens.

index=blend-sensitive-prod parsed.tenant=wellsfargo 
(kubernetes.container_name=disclosures OR kubernetes.container_name=blend-wellsfargo-mediumpriorityworkers)
(parsed.data.packageType=CLOSING_PACKAGE OR parsed.event.params.contextSelectors.packageType=CLOSING_PACKAGE) 
(parsed.httpMethod=POST OR parsed.httpMethod=PATCH OR "In disclosures event handler" OR 
"Sending disclosures package status event to the events pipeline")  
(parsed.data.status=COMPLETED OR parsed.data.status=SIGNED OR parsed.data.status=CANCELLED OR parsed.data.status=VIEWED OR
parsed.event.params.value=COMPLETED OR parsed.event.params.value=SIGNED OR parsed.event.params.value=CANCELLED OR parsed.event.params.value=VIEWED) 
(parsed.data.applicationId=* OR parsed.loanId=*)
(parsed.originalUrl="/api/events/envelopes" OR parsed.originalUrl="/api/disclosures/*" OR "DisclosuresExternal.status")
| rex "(packageI....)(?P<Package_UUID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rex "(applicationId...|loanId...)(?P<Loan_UUID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| stats earliest(eval(if(searchmatch("SIGNED"), _time, none))) as Package_Signed, 
earliest(eval(if(searchmatch("VIEWED"), _time, none))) as Package_Viewed,
latest(eval(if(searchmatch("COMPLETED"), _time, none))) as Package_Completed,
latest(eval(if(searchmatch("CANCELLED"), _time, none))) as Package_Cancelled by Package_UUID Loan_UUID
| eval Duration_Secs=Package_Completed-Package_Signed
| eval Package_Completed=strftime(Package_Completed,"%m/%d/%y %H:%M:%S.%3N")
| eval Package_Cancelled=strftime(Package_Cancelled,"%m/%d/%y %H:%M:%S.%3N")
| eval Package_Signed=strftime(Package_Signed,"%m/%d/%y %H:%M:%S.%3N")
| eval Package_Viewed=strftime(Package_Viewed,"%m/%d/%y %H:%M:%S.%3N")
| eval HourAgo=relative_time(now(),"-4h")
| eval HourAgo=strftime(HourAgo,"%m/%d/%y %H:%M:%S.%3N")
| eval Duration_Mins=round(Duration_Secs/60,0)
| where Package_Signed!="" AND Package_Viewed!="" AND isnull(Package_Completed)
AND isnull(Package_Cancelled) AND Package_Signed<HourAgo AND Package_Viewed<HourAgo
| fields Loan_UUID, Package_UUID, Package_Signed, Package_Viewed, Package_Completed, Package_Cancelled

- **WF Submitted Apps Not Exported**
  Identifies apps that have not had the export completed event trigger with 4 hours of the app being submitted.

(index=blend-sensitive-mulesoft-prod OR index=blend-sensitive-prod) wellsfargo 
(service="blend-wells-cb-connector-api" OR service="blend-sys-api" OR service="blend-consumer-banking-prc-api" OR kubernetes.container_name=blend-wellsfargo-mediumpriorityworkers)
(message.statusCode=200 OR message.tracepoint=END OR message.payload.eventType="applicationUpdated" OR message.text="*PATCHING*" OR message.eventData.data.trigger.type=EXPORTED OR message.tracepoint="blend-api-post-status-updates" OR parsed.loanType="PERSONAL_LOAN")
| rex "(trackingId....|applications.|loanId...)(?P<Loan_UUID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| stats earliest(eval(if(searchmatch("Event received from Blend"), _time, none))) as Export_Triggered, latest(eval(if(searchmatch("Response from wellsFargo after loan export") OR searchmatch("Loan export successfully completed") OR searchmatch("export-status") OR searchmatch("Blend API Patch Loan LosId") OR searchmatch("EXPORTED") OR searchmatch("Update BONS status to SUCCEEDED") OR searchmatch("Loan is not eligible for TWN on submit"), _time, none))) as Export_Completed by Loan_UUID
| eval Export_Duration_In_Secs=Export_Completed-Export_Triggered
| eval Export_Completed=strftime(Export_Completed,"%m/%d/%y %H:%M:%S.%3N")
| eval Export_Triggered=strftime(Export_Triggered,"%m/%d/%y %H:%M:%S.%3N")
| eval HourAgo=relative_time(now(),"-1h")
| eval HourAgo=strftime(HourAgo,"%m/%d/%y %H:%M:%S.%3N")
| eval Export_Duration_In_Mins=round(Export_Duration_In_Secs/60,0)
| where Export_Triggered!="" AND (isnull(Export_Completed) OR Export_Duration_In_Mins>240) AND Export_Triggered<HourAgo
| fields Loan_UUID Export_Triggered Export_Completed Export_Duration_In_Mins

- **WF Package Completed Delays**
  Identifies if package events are taking longer than the average amount of time.

index=blend-sensitive-prod parsed.tenant=wellsfargo 
(kubernetes.container_name=disclosures OR kubernetes.container_name=blend-wellsfargo-mediumpriorityworkers)
(parsed.data.packageType=CLOSING_PACKAGE OR parsed.event.params.contextSelectors.packageType=CLOSING_PACKAGE) 
(parsed.httpMethod=POST OR parsed.httpMethod=PATCH OR "In disclosures event handler" OR 
"Sending disclosures package status event to the events pipeline" OR "Sending disclosures recipient status event to the events pipeline")  
(parsed.data.status=COMPLETED OR parsed.data.status=SIGNED OR parsed.data.status=CANCELLED OR parsed.data.status=VIEWED OR
parsed.event.params.value=COMPLETED OR parsed.event.params.value=SIGNED OR parsed.event.params.value=CANCELLED OR parsed.event.params.value=VIEWED) 
(parsed.data.applicationId=* OR parsed.loanId=*)
(parsed.originalUrl="/api/events/envelopes" OR parsed.originalUrl="/api/disclosures/*" OR "DisclosuresExternal.status")
| rex "(packageI....)(?P<Package_UUID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| rex "(applicationId...|loanId...)(?P<Loan_UUID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| stats earliest(eval(if(searchmatch("SIGNED"), _time, none))) as Package_Signed, 
earliest(eval(if(searchmatch("VIEWED"), _time, none))) as Package_Viewed,
latest(eval(if(searchmatch("COMPLETED"), _time, none))) as Package_Completed,
latest(eval(if(searchmatch("CANCELLED"), _time, none))) as Package_Cancelled by Package_UUID Loan_UUID
| eval Duration_Secs=Package_Completed-Package_Signed
| eval Package_Completed=strftime(Package_Completed,"%m/%d/%y %H:%M:%S.%3N")
| eval Package_Cancelled=strftime(Package_Cancelled,"%m/%d/%y %H:%M:%S.%3N")
| eval Package_Signed=strftime(Package_Signed,"%m/%d/%y %H:%M:%S.%3N")
| eval Package_Viewed=strftime(Package_Viewed,"%m/%d/%y %H:%M:%S.%3N")
| eval HourAgo=relative_time(now(),"-1h")
| eval HourAgo=strftime(HourAgo,"%m/%d/%y %H:%M:%S.%3N")
| eval Duration_Mins=round(Duration_Secs/60,0)
| where Package_Signed!="" AND Package_Viewed!="" AND (isnull(Package_Completed) OR Duration_Mins>45) 
AND isnull(Package_Cancelled) AND Package_Signed<HourAgo AND Package_Viewed<HourAgo
| fields Loan_UUID, Package_UUID, Package_Signed, Package_Viewed, Package_Completed, Package_Cancelled, Duration_Mins

- **WF Docusign Errors**
  Simple query that identifies Docusign errors.

index=blend-sensitive-prod wellsfargo "Error calling Docusign: error from service docusign GET https://na2.docusign.net/restapi" "parsed.level"=error | rex ".envelopes.(?P<Envelope_UUID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" | dedup Envelope_UUID | table _time Envelope_UUID parsed.tenant

- **WF ECONNREFUSED Errors**
  Simple query that identifies ECONNREFUSED errors which usually mean there is an outage.

index="blend-sensitive-prod" parsed.err.cause.code=ECONNREFUSED "https://lgrid-unifiedgw-ext.wellsfargo.com:443*" (wellsfargo OR wellsfargopilot) | stats count by parsed.err.message parsed.tenant

- **WF SFTP Failures**
  Simple query that identifies failures with SFTP.

index="blend-sensitive-prod" "Error: Failed sendToWellsSFTP" "kubernetes.container_name"="integrations-yma" "Failed SubmitLoan for MORTGAGE loan" | dedup parsed.message | table _time parsed.message | sort - _time

- **WF Mulesoft Handshake Errors**
  Identifies if we receive back any errors in our Mulesoft layer when trying to export to WF.

index="blend-sensitive-mulesoft-prod" wellsfargo "failed: Received fatal alert: handshake_failure." | rex "trackingId....(?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" | dedup loanID | table _time message.payload.errorDescription message.route message.text loanID

- **WF Decision Errors** 
  Identifies any decision response that is not 200.

index="blend-sensitive-prod" "* GET /api/wells-consumer-banking/personal-loans/*/decision" NOT 200 | dedup parsed.req.params.loanId | table _time parsed.req.params.loanId parsed.res.statusCode parsed.req.url

- **WF Retry Failures**
  Identifies if an app has failed more than 4 times when retrying export.

index=blend-sensitive-mulesoft-prod message.tracepoint="wellsfargo Retry scope" service=blend-wells-cb-connector-api
| rex "trackingId....(?P<loanId>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" 
| stats count, last(_time) as timestamp by loanId, message.tracepoint, message.payload.errorDescription
| rename message.tracepoint as scope, count as export_attempts, message.payload.errorDescription as error
| fieldformat timestamp=strftime('timestamp', "%c") 
| sort - export_attempts 
| search export_attempts > 4

- **WF PL Submit/Export**
  Identifies export failures in the Mulsoft layer.

index=blend-sensitive-prod PERSONAL_LOAN ("Webhook of type applicationExport received" OR "wells-consumer-banking:Submit" OR "applicationCompleted" OR "applicationExported") "kubernetes.container_name"="integrations-yma" (parsed.tenantId=wellsfargo OR parsed.tenantId=wellsfargopilot)
| rex "loanId.....(?P<Loan_UUID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})"
| eval B_Submitted=if(searchmatch("wells-consumer-banking:Submit"), 1, 0), L_Submitted=if(searchmatch("Webhook of type applicationExport received"), 1, 0), Exported1=if(searchmatch("applicationCompleted"), 1, 0), Exported2=if(searchmatch("applicationExported"), 1, 0)
| stats earliest(_time) as Export_Started_Time, latest(_time) as Export_Completed_Time, sum(eval(B_Submitted+L_Submitted)) as App_Submitted, sum(eval(Exported1+Exported2)) as Export_Completed, range(_time) as Export_Duration by Loan_UUID, parsed.tenantId
| rename parsed.tenantId as Environment
| fieldformat Export_Started_Time=strftime('Export_Started_Time',"%c")
| fieldformat Export_Completed_Time=strftime('Export_Completed_Time',"%c")
| eval Export_Duration=tostring(Export_Duration,"duration")
| search Export_Completed=0

- **WF Rate Check Timeouts**
  Identifies 504 errors when trying to hit the rate check API.

index="blend-sensitive-prod" "1046-126" "parsed.dd.service"="wells-consumer-banking" "POST https://api.wellsfargo.com/personal-loans/new-accounts/v1/rates/retrieve: 504"

- **WF Config Change**
  Simple query to identify if anyone has made changed to the WF production config.

index=blend*events kking LendingConfig MODIFIED wellsfargo | table body.user body.updatedAt body.tenants{} body.action body.name meta.env meta.service meta.type | sort - body.updatedAt

- **WF CORE Request Failures**
  Simple query that identifies failures when attempting to send a file to CORE.

index="blend-sensitive-prod" loanId "Putting file to Wells SFTP" | join parsed.traceId  [search index="blend-sensitive-prod" wellsfargo "CORE Request failed * Internal Server Error" | table parsed.traceId] | dedup parsed.loanId | table parsed.loanId parsed.traceId

- **WF/Bilt Outage Firewall**
  Simple query that helps identify a specific issue with the Bilt firewall.

index="blend-sensitive-prod" "Terms & Conditions error" bilt-wf | rex "for application (?P<loanID>\w{8}\S\w{4}\S\w{4}\S\w{4}\S\w{12})" | dedup loanID | table _time loanID parsed.message parsed.tenant | sort - _time

- **CRV Testing**
  Query to identify the activity of the API user tied to WF CRV.

index=blend-internal-beta (813a9852-dddf-4f04-8001-9ee24f16c3e6 OR plc12@wellsfargo.com OR salesrcp00001.rcplast00001@wellsfargo.com OR plc13@wellsfargo.com) "kubernetes.container_name"="blend-wellsfargo-web" NOT "/api/external/loans" | table _time parsed.httpMethod parsed.message parsed.tenant parsed.originalUrl | sort - _time

- **WF Create Lender Failures**
  Query that identifies any create lender user failures.

index="blend-sensitive-prod" "Error: A borrower user with the email *@wellsfargo.com* already exists." parsed.message="Failed to create lender" | dedup parsed.err.externalMessage.error | table _time parsed.err.externalMessage.error parsed.message parsed.tenant