---
layout: post
title: Certificate Expiration posts to Teams
subtitle: In this one, I kept it simple....
tags: [Teams, Sharepoint, Flow, PowerAutomate]
comments: true
---

With the recent Teams outage many saw, it was revealed that the cause was the expiration of a certficate. It seems like such a small thing, but it’s very important in secure environments to not let certificates expire. This was something we identified a year or so back, and I started on a journey to look at a company wide solution for tracking upcoming internal certificate expirations. Suffice to say, I bit off more than I could chew and made it overly complicated. The group in charge of the Certificate Authorities also happened to find a much better way to monitor and track it!

Still, our team has services that we maintain that have certificates tied to functionality and we still wanted our own reminder system so we can keep on top of our systems. Enter what was Flow, now Power Automate. The functionality is fairly straight forward. I’ll outline the setup of the Flow below. As a prerequisite, you should have a Sharepoint Online calendar ready to go. Preferably, this calendar will be used only for tracking these expirations in order to avoid false alerts on the Teams channel.

These are how I’ve got our calendar events setup in order to show the end result message in Teams:

| Sharepoint Calendar Event | Teams Message Card |
| :------ |:--- | :--- |
| Event Title | Certificate name (usually as it shows in certificate store) |
| Date (set to all day event) | Expiration date |
| Description | Description – any extra info about the cert. |
| Created by | Created by |

Here is a brief overview of what the Flow will look like once completed.

![overview](/img/certexpteams/overview.png)

The first step will be to set your recurrence. We opted for a daily runtime of 10am.
Next, choose the Get Items action for Sharepoint. You will need the site address and the name of the calendar for this. This query below is what pulls any upcoming items in the next 30 days.

```
EventDate ge '@{convertTimeZone(utcNow(),'UTC','Central Standard Time','yyyy-MM-dd')}' and EventDate lt ' @{addDays(convertTimeZone(utcNow(),'UTC','Central Standard Time','yyyy-MM-dd'),30,'yyyy-MM-dd')}'
```
![cal](/img/certexpteams/cal.png)

We are going to initialize a variable that will be the table of upcoming certificate expirations. Name it whatever you like, but keep track of the name as you will need it elsewhere. Here is the initial http code for the table.

```
<table><thead><tr><th>Certificate</th><th>Expiration</th><th>Description</th><th>Created by</th></tr></thead><tbody>
```
![initvar](/img/certexpteams/initvar.png)

Now we need a condition. This will check the output from the Get Sharepoint Items step and if it is empty, discontinue the flow. Otherwise, it continues.

```
@not(empty(body('Get_expiration_dates_within_next_30_days')?['value']))
```

![condition1](/img/certexpteams/condition1.png)

In our Yes section, we will create an Apply to each, and when we finish it will look like this.

![applyto](/img/certexpteams/applyto.png)

First step within the Apply to each will be another condition. This condition will be to check how many days until the event is going to occur, from the day when the run occurs. My conditions are to continue if the alert is 30 days out, 10 days out, and finally 5 days or less (meaning it will post daily to when the calendar appointment is 5 days away). Here is the formula code to determine how many days away the event is.

```
div(sub(ticks(item()?['EndDate']),ticks(formatDateTime(utcNow(),'yyyy-MM-dd'))),864000000000)
```

![condition2](/img/certexpteams/condition2.png)

Our yes condition will look like this. The first 4 steps are just utilizing the Compose action under Data Operations as a way of formatting the data for the final step where we add a row of data to the table we started back at the start of the flow. Here is the code for formatting the expiration date in the first step.

```
@{formatDateTime(items('Apply_to_each')?['EndDate'],'MM-dd-yyyy')}
```

This is the code in the final step, Append to string variable action.

```
@{outputs(‘FormatCertName’)}@{outputs(‘FormatExpireDate’)}@{outputs(‘FormatDesc’)}@{outputs(‘FormatCreatedBy’)}
```

![yescondition](/img/certexpteams/yescondition.png)

Outside of the yes condition, we will be doing one more Append to string variable step. This will close up the table after each row has been added to the table we are storing expiration data in.

```
</tbody></table>
```

One final condition is setup. This one just checks the length of the ExpirationData variable which holds the table. As long as the length is longer than 134, then there is data in the table and it is ready to be posted to teams. I put this in as a final stop check to make sure we are only ever posting to teams when we have data to show.

```
length(variables('ExpirationData'))
```

![condition3](/img/certexpteams/condition3.png)

Within the yes condition, choose to post to a webhook. You will need to create a new webhook connector in teams (or you can use an existing one) and have the url it supplies handy for this. Be sure to set your content type as shown. Finally the body will contain the message card for all the info. Here is the code from mine.

```json
{
  "@context": "https://schema.org/extensions",
  "@type": "MessageCard",
  "themeColor": "0072C6",
  "title": "Certificate Expiration Alert",
  "text": "The following certificates are expiring in 30 days or less.@{variables('ExpirationData')}"
}
```

![webhook](/img/certexpteams/webhook.png)

Our end result when posted to a Teams channel, will look something like this.

![certteamscard](/img/certexpteams/certteamscard.png)

This flow could be modified to serve other purposes as well, even just a general event reminder for a group. Adjusting the days before an event could allow you to set it up as a day before reminder system for upcoming events, conferences, etc. that the group might be involved in.
