# Text-Message-Analysis

<!-- TOC -->
* [Text-Message-Analysis](#text-message-analysis)
  * [Libraries Used](#libraries-used)
  * [text-parser.ipynb](#text-parseripynb)
  * [data-cleaning.ipynb](#data-cleaningipynb)
  * [Analyzing.ipynb](#analyzingipynb)
    * [Finding Average Read Time](#finding-average-read-time)
    * [Finding Average Response Time](#finding-average-response-time)
    * [Total Messages Sent](#total-messages-sent)
    * [Total Characters Used](#total-characters-used)
    * [Total Number of Words](#total-number-of-words)
    * [Total Number of Unique Words](#total-number-of-unique-words)
    * [Total Number of Attachments](#total-number-of-attachments)
    * [Total Number of Double Texts](#total-number-of-double-texts)
    * [Heatmap of Day of Week vs Time of Day](#heatmap-of-day-of-week-vs-time-of-day)
    * [Heatmap of Month vs Year](#heatmap-of-month-vs-year)
    * [Heatmap of Month vs Day](#heatmap-of-month-vs-day)
* [Issues and Future Updates](#issues-and-future-updates)
* [Getting Your Own Results](#getting-your-own-results)
  * [Tools Used](#tools-used)
  * [Instructions For Using](#instructions-for-using)
<!-- TOC -->

## Libraries Used
- lxml
- Pandas
- re

## text-parser.ipynb
This file's main goal and task is to parse the html file that was created from the [imessage-exporter](https://github.com/ReagentX/imessage-exporter) tool.

After loading in the lxml library and opening the html file, I needed to actually parse the html with the lxml library. To do so, I read in all the lines and then fed them to the parser like so:

```python
with open(filename, 'r') as file:
    lines = file.readlines()
    string = ''.join(lines)
    html = lxml.html.fromstring(string)
```

After parsing, I created some helper functions that parse each element of the file that I wanted to analyze.
Here is an example of one of the functions:

```python
def getTimeStamp(element):
    els = element.find_class('timestamp')

    if els:
      return els[0].text_content()
    else:
      return ''
```

In total, there are 9 of the functions and the elements they parse are:
- Timestamp
- Sender
- Message
- Reaction
- Edited Element
- Attachments
- Attachment Links
- The Reply Anchor
- App Sent

Then I created a Message class to better structure each element like so:

```python
class Message:
    def __init__(self, timestamp, sender, message, reaction, edits, attachmentLinks, replyAnchor, appSent):
        self.timestamp = timestamp
        self.sender = sender
        self.message = message
        self.reaction = reaction
        self.edits = edits
        self.attachmentLinks = attachmentLinks
        self.replyAnchor = replyAnchor
        self.appSent = appSent
```

Now, all it takes is a simple for loop combined with the helper functions to get all the messages in the file

```python
messages = html.find_class('message')

message_list = []

for message in messages:
    message_list.append(Message(
        getTimeStamp(message),
        getSender(message),
        getMessage(message),
        getReaction(message),
        getEditedElement(message),
        getAttachmentLinks(message),
        getReplyAnchor(message),
        getAppSent(message)
        ))
```

From here the data was imported into a Pandas Dataframe and exported to a csv to allow for other files to load the data

```python
df = pd.DataFrame(x.toDict() for x in message_list)
df.to_csv('messages.csv', index=False)
```

## data-cleaning.ipynb

I wanted this file to effectively split the timestamp column that was extracted into easy to use columns. Specifically:
- Date
- Time
- Read By Time
- Read Time in Seconds

After loading the csv data into a dataframe, I created a function that uses regex to extract the separate elements of the time stamp. The function is as follows:

```python
def SeparateTimestamp(timestamp):
    date = re.search(r'(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)[\s]\d{2,4},\s\d{4}', timestamp)
    time = re.search(r'\d{1,2}:\d{2}:\d{2} (?:AM|PM)', timestamp)
    readTimeTemp = re.search(r'\([^)]*\)', timestamp)
    
    readTime = None
    if readTimeTemp is not None:
        readTime = readTimeTemp.group(0)
    
    return date.group(0), time.group(0), readTime
```

Then with these few lines, I had the column split up

```python
df['timestamp'] = df['timestamp'].apply(lambda x: SeparateTimestamp(x))
df[['date', 'time', 'read_time']] = df['timestamp'].apply(pd.Series)
df.drop(columns=['timestamp'], inplace=True)
```

The ```read_time``` column is now a string of hours, minutes, and seconds. It looks like ```2 hours, 3 minutes and 2 seconds```.
So I created a function to extract that into one singular integer value like so:

```python
def ExtractTime(string):
    time_patterns = {
        'hour': re.compile(r'(\d+)\s*hour'),
        'minute': re.compile(r'(\d+)\s*minute'),
        'second': re.compile(r'(\d+)\s*second')
    }
    
    time_dict = {}
    for unit, pattern in time_patterns.items():
        match = pattern.search(string)
        if match:
            time_dict[unit] = match.group(1)
    return time_to_seconds(time_dict)
    
def time_to_seconds(time_dict):
    seconds = 0
    if 'hour' in time_dict:
        seconds += int(time_dict['hour']) * 3600
    if 'minute' in time_dict:
        seconds += int(time_dict['minute']) * 60
    if 'second' in time_dict:
        seconds += int(time_dict['second'])
    return seconds

df['read_time_in_seconds'] = df['read_time'].apply(lambda x: ExtractTime(x) if (x != None) else None)
```

Lastly, before exporting this cleaned data to a csv, I needed to update the ```sender``` column. This column had 2 values "Me" and a phone number representing my wife.

```python
df['sender'] = df['sender'].apply(lambda x: 'Me' if x == 'Me' else 'My Wife')
```

## Analyzing.ipynb
There were many goals and statistics I wanted to see within our messages. I wanted to see:
- Average Read Time
- Average Response Time
- Total Messages Sent
- Total Characters Sent
- Total Words Sent
- Total Unique Words Used
- How Many Attachments Sent
- How Many Double Texts
- I also wanted to see some heatmaps comparing some data points:
  - Day of Week vs Time of Day Heatmap
  - Month vs Year Heatmap
  - Month vs Day Heatmap

### Finding Average Read Time
There is one glaring issue with finding average read times. My wife hasn't always had her read receipts on so there was not always a way for my phone to tell when she read a message.
Now, for my end of the read times, that didn't matter because my phone always stored how long it took me to read each message. So for this data point I will only be able to accurately get my 
data and not my wife's. This is the only data point thus far that has this case.

Because in ```data-cleaning.ipynb``` we did the heavy lifting in filtering the read time into an integer, it made it super easy to get the average. I just needed to filter the dataframe to only me and get the mean like so:

```python
me_df = df[df.sender == 'Me']
me_df['read_time_in_seconds'].mean()
```

I found that my average read time for my wife's messages is **4.41 seconds**

### Finding Average Response Time
Here, I wanted to find how long it took my wife and I to actually respond to each other. I first needed to find the difference in time from one message to the next. To do this, I used Pandas handy ```diff()``` function that does most of the work for me.
The problem I had here, was that since I had removed the timestamp column, I couldn't use that function immediately. I effectively had to recreate that column like ```pd.to_datetime(calc_df['date'] + ' ' + calc_df['time'])```. Afterwards I was able to
create a ```time_difference``` column using the ```diff()``` function like so:

```python
calc_df['time_difference'] = calc_df['timestamp'].diff()
```

This created an easy to use difference column. Next all I had to do was separate my wife and I and run a mean function on the columns to get my averages.

```python
me_df = calc_df[calc_df['sender'] == 'Me']
wife_df = calc_df[calc_df['sender'] == 'My Wife']

my_average = me_df['time_difference'].mean()
wife_average = wife_df['time_difference'].mean()
combined_average = (wife_df['time_difference'].mean() + me_df['time_difference'].mean()
```

My results were:
- **Me:** 16 Minutes and 53 Seconds


- **My Wife:** 26 Minutes and 58 Seconds 


- **Combined:** 21 Minutes and 55 Seconds


### Total Messages Sent
To get the total amount of messages sent, it was a simple process of again splitting the dataframe between my wife and I and then running a ```count()``` function on both dataframes to get the results.

```python
me_df = df[df['sender'] == 'Me']
wife_df = df[df['sender'] == 'My Wife']

my_count = me_df['message'].count()
wife_count = wife_df['message'].count()
```

My results were:
- **Me:** 33,498 messages


- **My Wife:** 31,875 messages 


- **Combined:** 65,373 messages

### Total Characters Used
To get the total amount of characters used, I simply, using the already split dataframes, just needed to get the length of each message and sum them all up.

```python
my_count = me_df['message'].apply(lambda x: len(str(x))).sum()
wife_count = wife_df['message'].apply(lambda x: len(str(x))).sum()
```

My results were:
- **Me:** 1,421,613 characters


- **My Wife:** 1,237,373 characters 


- **Combined:** 2,658,986 characters


### Total Number of Words
This one was a little trickier than the last 2 but I wanted to get how many words we each used. To accomplish this, I first needed to cast my message as a string like ```str(x)``` to make sure it would always read as a string. I then needed to split the string, using spaces as the separator, into an array by using the split method like ```x.split()```. I then needed to get the length of that array which would equal the amount of words in that message like ```len(x)``` and then use the ```sum()``` method to add all the values up to get my answer.
I condensed these steps into one line and used them like so:

```python
my_count = me_df['message'].apply(lambda x: len(str(x).split())).sum()
wife_count = wife_df['message'].apply(lambda x: len(str(x).split())).sum()
```

My results were:
- **Me:** 291,100 words


- **My Wife:** 258,687 words 


- **Combined:** 549,787 words

### Total Number of Unique Words
To get the total number of unique words, it's almost the exact same as the getting total number of words but with a few changes. Firstly, we aren't going to use the ```len()``` method here in the same way we just used it. We first need to make sure all values in the list are unique. So instead, after we use the ```split()``` function we immediatley can use the ```sum()``` function to append the arrays onto each other.
After the arrays are combined, we use Python's ```set()``` method to create a list of unique values. After we have are unique values we simply run a ```len()``` function on it to get our answer. After getting the unique values for both my wife and I, I run a union on the 2 sets to get a final combined unique set of words.

```python
my_words = me_df['message'].apply(lambda x: str(x).split())
my_word_list = my_words.sum()

unique_list = set(my_word_list)

wife_words = wife_df['message'].apply(lambda x: str(x).split())
wife_word_list = wife_words.sum()

wife_unique_list = set(wife_word_list)

len(unique_list)
len(wife_unique_list)

combined = set(wife_unique_list | unique_list)
len(combined)
```

My results were:
- **Me:** 17,299 words


- **My Wife:** 16,479 words 


- **Combined:** 25,953 words

### Total Number of Attachments
I wanted to get how many attachments we have sent. This includes all kinds of attachments like shared apps, images, videos, gifs, and so on. The backup file I exported did not contain the actual content of the attachments so I have elected, for now, to only count the total number of attachments.

To get this answer, I filtered my wife and I's separate dataframes to only include rows that have at least one attachment. After this I combined all the arrays of attachemnt links into one and got the length of that array.

```python
my_attachments_df = me_df[me_df['attachmentLinks'].notna()]
my_attachments = my_attachments_df['attachmentLinks'].sum()
```

After doing this for my wife as well, I got these results:
- **Me:** 22,440 attachments


- **My Wife:** 31,484 attachments


- **Combined:** 53,924 attachments

### Total Number of Double Texts
A double text is where the same person sends 2 texts in a row without the other person responding. To get this, I created a new dataframe with the ```sender``` and ```message``` column. After this step, I created a new column that is a boolean value. The value will be ```True``` if the previous row in the dataframe contained the same sender (meaning that person double texted) and ```False``` if the previous row was another sender.
Then all I had to do was filter for what values were true and ran the ```count()``` function on it.

```python
double_texts_df = df[['sender', 'message']]

double_texts_df = double_texts_df.assign(double_text=double_texts_df.sender.eq(double_texts_df.sender.shift()))

my_double_texts = double_texts_df.loc[(double_texts_df.sender == 'Me') & (double_texts_df.double_text)].double_text.count()
wife_double_texts = double_texts_df.loc[(double_texts_df.sender == 'My Wife') & (double_texts_df.double_text)].double_text.count()
combined_double_texts = double_texts_df.loc[(double_texts_df.double_text)].double_text.count()
```
My results were:
- **Me:** 6,533 double texts


- **My Wife:** 5,065 double texts


- **Combined:** 11,598 double texts


### Heatmap of Day of Week vs Time of Day
Creating heatmaps was a little bit of an involved process with this data. I first filtered the dataframe down to only the ```date``` and ```time``` columns. Afterwards, I converted the ```time``` column into 24 hour format to make calculations easier.

```python
converted_df = calc_df[['date', 'time']]
converted_df = converted_df.assign(time=pd.to_datetime(converted_df['time'], format="%I:%M:%S %p"))
```

This is the dataframe outputted by the code:

|    | date         | time                |
|----|--------------|---------------------|
|  0 | Sep 28, 2021 | 1900-01-01 10:21:18 |
|  1 | Sep 28, 2021 | 1900-01-01 10:21:30 |
|  2 | Sep 28, 2021 | 1900-01-01 10:21:46 |
|  3 | Sep 28, 2021 | 1900-01-01 10:22:20 |
|  4 | Sep 28, 2021 | 1900-01-01 10:22:54 |

I then created a new column that showed what weekday that message was sent on using this code:
```python
converted_df = converted_df.assign(day_of_week=pd.to_datetime(converted_df['date']).dt.day_name())
```

Now that I had the day of the week, the actual date did not matter, so I got rid of that column.

I then wanted to bin the ```time``` column into 3 hour increments. I used the ```pd.cut()``` function to accomplish this and fed it only the hour from the ```time``` column, labels to use for the binning, the actual bin parameters, and changed ```right``` to false so there was no overlap.

```python
heatmap_df = converted_df[['time', 'day_of_week']]

bins = [0, 3, 6, 9, 12, 15, 18, 21, 24]
labels = ['12AM-3AM', '3AM-6AM', '6AM-9AM', '9AM-12PM', '12PM-3PM', '3PM-6PM', '6PM-9PM', '9PM-12AM']
heatmap_df['Time Bin'] = pd.cut(heatmap_df.time.dt.hour, bins, labels=labels, right=False)
```

Now I created a new ```Hour``` column in the dataframe because I did not need accuracy down to the minute. I only needed the hour. I then removed the ```time``` column as it was not necessary.

```python
heatmap_df['Hour'] = heatmap_df['time'].dt.hour
heatmap_df = heatmap_df[['day_of_week', 'Hour', 'Time Bin']]
```

This resulted in the following dataframe:

|    | day_of_week   |   Hour | Time Bin   |
|----|---------------|--------|------------|
|  0 | Tuesday       |     10 | 9AM-12PM   |
|  1 | Tuesday       |     10 | 9AM-12PM   |
|  2 | Tuesday       |     10 | 9AM-12PM   |
|  3 | Tuesday       |     10 | 9AM-12PM   |
|  4 | Tuesday       |     10 | 9AM-12PM   |

Now all I needed to do was create the actual heatmap. Firstly, I needed to create a pivot table with the index being ```day_of_week```, the columns being ```Time Bin```, the values being ```Hour```, and with the aggregate function being ```count```.

```python
pivot = heatmap_df.pivot_table(index="day_of_week", columns="Time Bin", values="Hour", aggfunc='count', observed=True)
```

To sort the index (```day_of_week```) into the arrangement I wanted, which is in order with the week starting on Sunday, I needed to make the index a categorical index and sort it.

```python
pivot.index = pd.CategoricalIndex(pivot.index, categories=['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'])
pivot.sort_index(level=0, inplace=True)
```

With the pivot table all set up, I used seaborn to generate a heatmap of the data.

```python
ax = sns.heatmap(pivot, cmap="gnuplot", cbar=False, linewidths=0.5, square=True)

ax.set(xlabel='Time of Day', ylabel='Day of Week', title='Day of Week vs Time of Day Heatmap')
plt.tight_layout()
plt.savefig('images/day_of_week_heatmap.png')
```

The resulting heatmap:

![](/images/day_of_week_heatmap.png "Heatmap of Days of Week vs Time of Day")

### Heatmap of Month vs Year
It was almost the exact same process as the previous heatmap but I instead got the ```year``` and ```month``` from the ```timestamp```.

```python
mo_df = calc_df[['date', 'message']]

mo_df['month'] = pd.to_datetime(mo_df['date']).dt.month_name()
mo_df['year'] = pd.to_datetime(mo_df['date']).dt.year

mo_df = mo_df[['month', 'year', 'message']]
```

With almost the exact same code as before I created a pivot table, sorted it, and generated a heatmap with the data.

```python
mo_pivot = mo_df.pivot_table(index='month', columns='year', values='message', aggfunc='count', observed=True)

mo_pivot.index = pd.CategoricalIndex(mo_pivot.index, categories=['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'])
mo_pivot.sort_index(level=0, inplace=True)

ax = sns.heatmap(mo_pivot, cmap="gnuplot", linewidths=0.5, square=False, annot=True, fmt='.2f')

ax.set(xlabel='Year', ylabel='Month', title="Month Over Year Heatmap")
plt.tight_layout()
plt.savefig('images/month_over_year_heatmap.png')
```

The resulting heatmap:

![](/images/month_over_year_heatmap.png)

### Heatmap of Month vs Day
The last heatmap I wanted to create and see was the number of text messages sent each day of the week for each month.

It was again the same process as before with just editing what columns I am feeding to the pivot table.

```python
md_df = pd.DataFrame(calc_df['date'])

md_df['month'] = pd.to_datetime(md_df['date']).dt.month_name()
md_df['day_of_week'] = pd.to_datetime(md_df['date']).dt.day_name()

md_pivot = md_df.pivot_table(index='month', columns='day_of_week', values='date', aggfunc='count', observed=True)

md_pivot.index = pd.CategoricalIndex(md_pivot.index, categories=['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'])
md_pivot.sort_index(level=0, inplace=True)

ax = sns.heatmap(md_pivot, cmap="gnuplot", linewidths=0.5, square=False, annot=True, fmt='.2f')

ax.set(xlabel='Day', ylabel='Month', title="Month Over Day Heatmap")
plt.tight_layout()
plt.savefig('images/day_over_month_heatmap.png')
```

The resulting heatmap:

![](/images/day_over_month_heatmap.png)

# Issues and Future Updates
- One issue I spotted pretty quickly was for the unique words. Because the split function only splits based on spaces, some words end up getting a trailing comma or period to the end of them. This would cause the ```set()``` method to not be able to catch the same word in some instances. For example, it would see ```Hello``` and ```Hello,``` as different words when they are clearly not.
  - I plan to fix this at some point possibly by stripping all messages of punctuation before splitting in order to try and account for this problem.
- I would like to create a front end website and dashboard that shows this data and allows for me to generate up to date graphs.
  - Also want to make this able to filter timelines.

# Getting Your Own Results
## Tools Used
- [imessage-exporter](https://github.com/ReagentX/imessage-exporter)

## Instructions For Using
1. Install [imessage-exporter](https://github.com/ReagentX/imessage-exporter).
2. Get an unencryped backup of your iPhone onto your computer.
3. Run the command ```$ imessage-exporter -f html -p ~/path_to_iphone_backup -a iOS -o output_path``` to export your iMessage data into html format.
4. Change the file location of your html file in ```text-parser.ipynb``` and run all the cells.
5. Next point ```data-cleaning.ipynb``` to the new csv file that was just created and run all cells.
6. Now you can use the cleaned csv file and run all cells in ```Analyzing.ipynb``` to get the same data points I got on your own messages.



   
