+++
author = "Richmond Liew"
title = "PyCollect"
date = "2020-08-30"
description = "An automation project"
tags = [
    "automation", "convenience", "Python"
]
+++

tl;dr: Like a true engineur, I spent many hours automating something that could've been done manually in 5 mins. [Source](https://github.com/liewrichmond/pyCollect2.0)
<!--more-->
For those who don't know, I own a cozy 3 bedroom apartment in the Roxubry/Jamaica Plain area and rent 2 of the rooms out. The way I calculate the rent due each month is pretty standard. I charge a flat rate for each room and split the utilties with my roommates/tenants each month. Although this only takes ~5 minutes a month, it tends to slip my mind and having to dig through my email/utilities accounts to find the monthly amounts can get really annoying... So why not automate it? Btw, the code for this projects is open sourced on Github [here](https://github.com/liewrichmond/pyCollect2.0).

### Setting things up
As always, the first step to automating anything is simplifying the problem. Instead of having to run through each portal and scrape the data etc., I made sure that each utilities bill was synced to my email account and signed up for paperless billing (go green, baby). Along with that, I created a Utilities label and filtered all my bills into that label for easy searching.

![example Image](/images/UtilitiesLabel.png)

Now we can finally have some fun with Python. The flow of the program is pretty straightforward. The steps are:
1. Connect to Gmail
2. Get mail from utilities label
3. Get dollar amounts from mail
4. Request Amounts from roommates

### The Code
I'm only going to talk about the interesting portions of the program.

To connect to the gmail API, I just used the starter script from the documentation. In addition to this, I needed a couple extra helper functions so I wrapped it into a class like so:
```python
class Calculator():
...
      def connect_to_gmail(self):
        # The file token.pickle stores the user's access and refresh tokens, and is
        # created automatically when the authorization flow completes for the first
        # time.
        if os.path.exists('token.pickle'):
            with open('token.pickle', 'rb') as token:
                self.creds = pickle.load(token)
        # If there are no (valid) credentials available, let the user log in.
        if not self.creds or not self.creds.valid:
            if self.creds and self.creds.expired and self.creds.refresh_token:
                self.creds.refresh(Request())
            else:
                flow = InstalledAppFlow.from_client_secrets_file(
                    'credentials.json', SCOPES)
                self.creds = flow.run_local_server(port=0)
            # Save the credentials for the next run
            with open('token.pickle', 'wb') as token:
                pickle.dump(self.creds, token)
        self.service = build('gmail', 'v1', credentials=self.creds)
...
```
With access to my inbox, I could start looking through my messages. Since I only wanted to get the bills from the past month, I had to create a date query and attach to to my requests.
```python
...
      def get_past_month_date(self, year, month):
          if(month == 1):
              return datetime.date(year-1, 12, 1)
          else:
              return datetime.date(year, month, 1)
  
      def get_mail_messages(self,labels):
          today = datetime.date.today()
          searchDate= self.get_past_month_date(today.year, today.month)
          dateQuery = 'after:{searchDate}'.format(
              searchDate=str(searchDate).replace('-', '/'))
          utilities = self.service.users().labels().get(
              userId='me', id=self.label_id).execute()
          messages = self.service.users().messages().list(
              userId='me', labelIds=self.label_id, q=dateQuery).execute()
          return messages
...
```
Now that I finally have my messages, all I need to do is parse them for the dollar amounts and send a Venmo request. There's just one more thing. The messages are base64 encoded. No worries, though, by running it through Python's built in base64 decoder, I was able to get the plaintext. I thought I'd be able to easily search through to get the dollar amounts. However, The emails aren't all that pretty in non-html form. They usually come with images, formatting etc. which makes them kinda ugly.

This is where Regex comes in! By creating a regex search pattern, I was able to loop through the messages and get the dollar amounts fairly easily. The regex pattern only looks for a '$' followed by numbers and a period and terminates once it hits a space.
```python
...
      def compile_regex(self):
          self.regexQuery = re.compile(r'([\$]+[\060-\071.]*)')

      def get_dollar_amount(self, messages):
        total = 0
        for message in messages['messages']:
            reply = self.service.users().messages().get(
                userId='me', id=message['id'], format='raw').execute()
            msg_str = base64.urlsafe_b64decode(
                reply['raw'].encode("utf-8")).decode("utf-8")
            found = self.regexQuery.search(msg_str)
            amount = float(found.group(0).split('$')[-1])
            total += amount
        return total
...
```
I wrapped both the `get_mail_messages` and `get_dollar_amount` into a nice little method which looks pretty clean
```python
      def get_utilities_total(self):
          # Call the Gmail API
          results = self.service.users().labels().list(userId='me').execute()
          labels = results.get('labels', [])

          messages = self.get_mail_messages(labels)
          total = self.get_dollar_amount(messages)
          return total

```
And now comes the final piece which is finally collecting the damn rent to pay the bills. The unofficial Venmo API makes this fairly easy to do by just calling a `request_money()` method
```python
...
      def send_request(self):
          venmo = Client(
              access_token=self.venmo_token)

          total = self.get_utilities_total()
          amountDue = total/(len(self.tenants)+1)
          today = datetime.date.today()
          prevMonth = self.get_past_month_date(today.year, today.month)
          requestNote = months[prevMonth.month] + " Utilities"

          for tenant in self.tenants:
              tenantUsername = venmo.user.search_for_users(query=tenant)[0]
              venmo.payment.request_money(
                  amount=amountDue, note=requestNote, target_user=tenantUsername)
...
```

### Hiding Sensitive Information
The way I initially wrote the program, I just had everything wirtten inline. This made pretty sketchy to have it open sourced since I didn't want to put my Venmo token out on the internet. So, I created a little `secrets.json` file to keep my secrets like tenant Venmo usernames and Venmo access tokens etc. This file looks like
```json
{
    "labelId":"labelStr",
    "venmoAccessToken":"venmoTokenStr",
    "venmoUsernames":["venmoUsername1", "venmoUsername2"]
}
```
and is included in the main repo as `sampleSecrets.json`. If I've made a mistake and accidentally included this information somewhere in the repo, please don't hack me and kindly send me an email to git gud thanks :)