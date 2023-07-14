# Story-Issue-MigrationBatch-Failed
Story-Issue-MigrationBatch-Failed

## Situation

- OnPrem AD synchronized with Azure AD tenant with AADConnect
- AD User created OnPrem, synchronized with Azure AD
- Enabled Remote Mailbox from OnPrem (using ```Enable-RemoteMailbox -Identity "User A" -RemoteRoutingAddress "userA@tenantname.mail.onmicrosoft.com```)
- New-MigrationBatch from Exchange Admin Console from Exchange Online

 ## Issue

 The migration batch fails with the error ```RecipientNotFoundPermanentException```

 ## Troubleshooting steps

- first we checked if the Mail Enabled User located OnPrem was syncrhonized correctly with the Mailbox Enabled User in Exchange Online.
For that we checked the ImmutableID on the Azure AD object, and compared it with the ObjectID on the OnPrem object.
Steps to do that are the following:
  - Export the Azure AD user ImmutableID property (it is in Base64)
  - Get the OnPrem AD user ObjectID and convert it to Base64
  - Compare the ImmutableID with the Base64 converted ObjectID

In our case these were the same => after confirming with the AADConnect sync logs, we confirmed the object was correctly synchronized.

- Then we removed the MigrationBatch and created another one, and monitored the user move using the following:

```powershell
Get-MigrationUser | Where-object {$_.Identity -like "*User_Address_substring*"} | Get-MigrationUserStatistics | Select-Object identity,batchid,error,errorsummary, TotalItemsInSourceMailboxCount, SyncedItemCount, percentagecomplete
```

- then after some research we found that it could be that the ExchangeGUID could be unpopulated on the OnPrem side (since the O365 mailbox was enabled remotely from OnPrem)

1. Run the below from the Exchange Management Shell on the **on-premises server**

```powershell
Get-RemoteMailbox <alias of cloud mailbox to move> | Format-List ExchangeGUID
```

> NOTE: If the ExchangeGUID property returns all zeros, the value isn't stamped on the on-premises remote mailbox.

2. Run the below from the Exchange Online Management Shell

```powershell
Get-Mailbox <MailboxName> | Format-List ExchangeGUID
```

3. From **Exhcange OnPrem**

```powershell
Set-RemoteMailbox <MailboxName> -ExchangeGUID <GUID>
```

4. Force or wait for DirSync

The above steps are taken from [this Microsoft Learn article](https://learn.microsoft.com/en-us/exchange/troubleshoot/move-mailboxes/migrationpermanentexception-when-moving-mailboxes)

## Try moving a mailbox again

Try a Migration Batch again, and monitor with the below again:

```powershell
Get-MigrationUser | Where-object {$_.Identity -like "*User_Address_substring*"} | Get-MigrationUserStatistics | Select-Object identity,batchid,error,errorsummary, TotalItemsInSourceMailboxCount, SyncedItemCount, percentagecomplete
```


