# Useful Queries

## Find out the users by role across accounts (AUTH)

Use case : Query all BCEID admins in BC Registries across all accounts.
Change role and login source accordingly to meet different requirements.
```
select 
	o.name as account_name, 
	u.first_name, 
	u.last_name, 
	u.username, 
	c.email 
from users u 
left join memberships m on m.user_id=u.id
left join contact_links cl on cl.user_id=u.id
left join contacts c on c.id = cl.contact_id
left join orgs o on o.id=m.org_id
where 
m.membership_type_code='ADMIN' 
and u.login_source='BCEID' 
and o.status_code='ACTIVE' 
and m.status = 1
```

## Find out the invoices and payment statuses.
```
select 
i.id invoice_id, ir.id inv_ref_id, r.id receipt_id, i.total, i.paid, i.created_on,  ir.status_code invoice_reference_status,  ir.invoice_number, i.invoice_status_code, ca.cfs_account, ca.cfs_party, ca.cfs_site
from invoices i
left join invoice_references ir on i.id = ir.invoice_id
left join payment_accounts pa on pa.id = i.payment_account_id
left join cfs_accounts ca on ca.account_id = pa.id
left join receipts r on r.invoice_id = i.id
where 
-- pa.auth_account_id in ('')
-- ca.cfs_account=''
i.business_identifier in ('NR 0001', 'CP 0001', 'BC 0001')
order by ca.cfs_account, invoice_number;
```

## Find invoices for EJV transactions for billable accounts
```

select 
pa.name, i.id invoice_id, i.total, i.paid, i.created_on,  i.corp_type_code, i.payment_method_code, i.invoice_status_code
from invoices i
left join invoice_references ir on i.id = ir.invoice_id
left join payment_accounts pa on pa.id = i.payment_account_id
left join cfs_accounts ca on ca.account_id = pa.id
left join receipts r on r.invoice_id = i.id
where 
pa.billable = true
and i.payment_method_code='EJV'
order by ca.cfs_account, pa.id;
```
### Group by account
```
select 
pa.name, pa.id, sum(i.total)
from invoices i
left join invoice_references ir on i.id = ir.invoice_id
left join payment_accounts pa on pa.id = i.payment_account_id
left join cfs_accounts ca on ca.account_id = pa.id
left join receipts r on r.invoice_id = i.id
where 
pa.billable = true
and i.payment_method_code='EJV'
group by pa.id;
```

## How to curl invoice details in CFS.
### First Generate a token.
```
curl -X POST https://cfs-prodws.cas.gov.bc.ca:7121/ords/cas/oauth/token -u cfs_client_id:cfs_lient_secret -H 'Content-Type:application/x-www-form-urlencoded' -d 'grant_type=client_credentials' -i
```
### Use the access_token from above response and query CFS.
Party number, account number and site number are in cfs_accounts table in pay-db.
Invoice number is in invoice_references table.

```
curl -X GET https://cfs-prodws.cas.gov.bc.ca:7121/ords/cas/cfs/parties/<cfs_party>/accs/<cfs_account>/sites/<cfs_site/invs/<inv_number>/ -H 'Content-Type:application/json' -H 'Authorization: Bearer <token>' -i
```

## Find tasks for the account
```
select t.id, t.status from tasks t left join orgs o on o.id = t.relationship_id 
where o.name like '%REDHEAD%';
```

## How to push a payment which is completed, but not reflected on pay side.
```
https://www.bcregistry.ca/business/auth/makepayment/117462/https%3A%2F%2Fwww.bcregistry.ca%2Fnamerequest%2Fnr%2F2189563%2F%3FpaymentId%3D114644
```


## Steps to follow for reconciliation errors
In some cases reconciliation may fail for some records, and would stop processing following lines. In that case run the below query to see which all invoices are still not processed.
```
select 
	i.id, i.invoice_status_code, i.payment_account_id, i.cfs_account_id, ir.invoice_number
from invoices i
left join invoice_references ir on i.id=ir.invoice_id
where i.invoice_status_code='APPROVED' and
ir.invoice_number in
(<HERE COPY ALL THE INVOICE NUMBERS FROM THE PAD SETTLEMENT FILE>);
```

OR just to get the invoice_number
```
select 
	distinct ir.invoice_number
from invoices i
left join invoice_references ir on i.id=ir.invoice_id
where i.invoice_status_code='APPROVED' and
ir.invoice_number in
(<HERE COPY ALL THE INVOICE NUMBERS FROM THE PAD SETTLEMENT FILE>);

```

Now investigate the issue; may be a data fix or code fix might be needed based on the issue. Then we need to upload the reconciliation file again for the failed entries. 
- Run the above query and find out the failed rows.
- Open the reconciliation CSV file; then remove all other rows which are sucess
- Save it with different name
- Upload to minio
- Then go to payment reconciliations pod terminal and execute below command;
```
./q_cli.py -f <FILE NAME in minio> -m '' -l ''
```


## How to find duplicate payment records

```
SELECT
  invoice_number,
  payment_account_id,
  payment_status_code,
  count(*),
  string_agg(id::text, ',')
FROM payments
GROUP BY
  invoice_number,
  payment_account_id,
  payment_status_code
HAVING count(*) > 1;
```

## Query for duplicate NSF invoices created.

```
SELECT
  payment_account_id,
  invoice_status_code,
  count(*),
  string_agg(id::text, ',')
FROM invoices
where corp_type_code='BCR'
GROUP BY
  payment_account_id,
  invoice_status_code
HAVING count(*) > 1;
```

## query invoice using a value from details column
select id, invoice_status_code,detail.label, detail.value from invoices,jsonb_to_recordset(invoices.details) as detail(label text, value text) where jsonb_typeof(details) ='array' and value = '<value>';


## Manually REMOVE NSF
To manually remove NSF below things have to happen;

1 - Update the site receipt_method to BCR-PAD Daily (execute from a pod)

```
curl -X PUT https://cfs-prodws.cas.gov.bc.ca:7121/ords/cas/cfs/parties/<cfs_party>/accs/<cfs_account>/sites/<cfs_site/ -H 'Content-Type:application/json' -H 'Authorization: Bearer <token>' -d '{"receipt_method": "BCR-PAD Daily"}' -i
```

2 - Update cfs_accounts status to ACTIVE (from FREEZE)

3 - Update orgs status to ACTIVE (in Auth database)





How to get CFS token
														      
``` 
curl -X POST https://cfs-systws.cas.gov.bc.ca:7025/ords/cas/oauth/token -H 'Authorization: Basic c2lnT2Vab0llbmNkTVk4OGFwS2FYUS4uOnZQXzV4NmNKMjFRMzRUdU9DYnFiZ1EuLg==' -H 'Content-Type: application/x-www-form-urlencoded' -d grant_type=client_credentials
														      
```
														  
														      

														      
														      
