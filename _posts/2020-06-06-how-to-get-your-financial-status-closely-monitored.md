---
title: How to get you financial status closely monitored 
date: 2020-06-07 21:25:00
categories:
 - Life Hack
tags: finance
---
This post is going to talk about how to monitor your financial status in a self-serve way with no privacy concerns.

## Background

After buying my first home, there is pressing feeling inside me that made myself feel obliged to have a better understanding of my financial situation. When you have to make some big transaction in your life, you probably have to do some napkin math to get to know how much money you have in total across all your banks & accounts. I feel this is such a simple question and could be useful to know all the time even I’m not gonna buy something big, but I actually have no idea how to get it easily since they are all over the place. 

I was told some third-party service can help to integrate all your bank accounts in one place and provides recommendations or insights for better financial planning. However, I am just too timid to give away my banking credentials to any third-party service. In the end, I decided to just make a simple tool for myself to monitor my banking status all together and only run locally on my laptop. Basically, the goals to build this tool are:
- Collect all my banking information together and present in one place;
- Provide some basic knowledge about my financial stats to understand my balance and purchasing behavior, and help saving and planning;
- Have no privacy concern;

Based on the motivation above, I start to look for the simplest and most secure way to implement this.  The workflow is actually quite straightforward which includes:

*Data Collection --> Data Processing --> Data Visualization*

I’ll explain how I did each part in the following sections. All the codes are written in Python.

## Data Collection
The first step is to get transaction level details from your online banking website. You can definitely download this manually every time, but it would be awesome to automate the whole thing. To collect the data, I used selenium which is a Python package can automate browsers. Since banks have different ways to manage your transactions, you probably need to write different scripts for each bank you are with. For example, here is some code to get your transaction data for RBC online banking:
```python
import os
from pathlib import Path
import shutil

from selenium.webdriver import Chrome
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import Select

# https://sites.google.com/a/chromium.org/chromedriver/downloads
# check the above link if the chrome driver mismatches the chrome version

url = 'https://www.rbcroyalbank.com/ways-to-bank/online-banking/index.html'

options = Options()

driver = Chrome(chrome_options=options, executable_path='your_chromedriver_path')
driver.get(url)
driver.implicitly_wait(5)

# login page
username = driver.find_element_by_id('K1').send_keys('your_card_number')
driver.implicitly_wait(2)
password = driver.find_element_by_name('Q1').send_keys('your_password')
driver.implicitly_wait(2)
login_button = driver.find_element_by_class_name('yellowBtnLarge').click()
driver.implicitly_wait(10)

#home page
product_and_service = driver.find_element_by_xpath('/html/body/div[1]/div/header/div[2]/div[1]/ul/li[1]/a').click()
driver.implicitly_wait(5)
account_serice = driver.find_element_by_xpath('/html/body/div[2]/div[4]/div[1]/div[2]/ul/li[2]/a').click()
driver.implicitly_wait(5)
download_transactions = driver.find_element_by_xpath('/html/body/div[1]/div[6]/div[1]/div[2]/ul/li[2]/ul/li[4]/a').click()
driver.implicitly_wait(5)

#download page
choose_csv = driver.find_element_by_xpath('//*[@id="Excel"]').click()
driver.implicitly_wait(5)
transaction_options_dropdown = Select(driver.find_element_by_xpath('//*[@id="transactionDropDown"]'))
driver.implicitly_wait(5)
transaction_options_dropdown.select_by_visible_text('All Transactions on File')
driver.implicitly_wait(5)
driver.find_element_by_xpath('//*[@id="id_btn_continue"]').click()
driver.implicitly_wait(15)


download_path = 'your_download_folder_path'
paths = sorted(Path(download_path).iterdir(), key=os.path.getmtime)
latest_file_path = str(paths[-1])
shutil.move(latest_file_path, 'path_your_want_to_organize_all_the_transaction_files')
```
You may see me using `find_element_by_xpath` a lot, and that is because I fall in love with this Chrome extension called [xPath Finder](https://chrome.google.com/webstore/detail/xpath-finder/ihnknokegkbpmofmafnkoadfjkhlogph?hl=en), which can help you find out the xPath for the selected element immediately. After I found it, I give up on using any other way to locate an element on a webpage since it can speed up the whole process 10x.

From the code above, you can see how easy it is to pull data from your bank by hitting the run button. And I believe you can figure out how to pull data for your own bank by using the similar idea. But be careful about how different banks organize your transactions. For instance, RBC allows you to download all your transactions in one CSV file but only for the past 6 months; While CIBC (another bank I’m with) lets you download all your historical transactions, but you need to download for each account you have respectively. 

## Data Processing

After you have all your transaction data in hand, you can start to organize your data in the ideal way you were always thinking about. I found there not a lot to share in this section since there could be so many ways to get this done, and people may have different expectations when organizing their own transactions. What I did for this part is basically cleaning up the transaction files from different banks by renaming some fields, concatenate description fields for some bank, formatting the fields I want to keep and union all the records together. I found the records for credit cards can be a bit different from the others that needs to be handled specifically. No matter how you want to do this, the goal is to make it ready to be used for the following visualization.

## Data Visualization

If you have Tableau, especially Tableau Individual license, I’ll strongly recommend you use Tableau and problem solved. For me, Tableau is good enough to make all charts I wanted in my mind with its beautiful charts. Alternatively, you can also use [Dash open source](https://dash.plotly.com/). In Dash’s webpage, it says “Dash apps run on your local laptop or workstation but cannot be easily accessed by others in your organization”, while, it’s perfectly for my use case since I don’t want to push anything to the cloud. To be more specific, 

*Dash is a productive Python framework for building web applications.*
*Written on top of Flask, Plotly.js, and React.js, Dash is ideal for building data visualization apps with highly custom user interfaces in pure Python. It's particularly suited for anyone who works with data in Python.*

I decided give Dash a try and so far it works pretty well. I can just make all kinds of transformation I wanted in Pandas Dataframe and pass it to Dash, and Dash can simply handle it without writing any Javascript. 
```python
curent_balance = raw_df['CAD'].sum()

def visualize():

    external_stylesheets = ['https://codepen.io/chriddyp/pen/bWLwgP.css']
    app = dash.Dash(__name__, external_stylesheets=external_stylesheets)

    colors = {
        'background': '#ffffff',
        'text': '#595959'
    }

    general_chart_layout = {
                    'plot_bgcolor':     colors['background'],
                    'paper_bgcolor':    colors['background'],
                    'font': {'color':   colors['text']},
                    'autosize':         True,
    }

    app.layout = html.Div(style={'backgroundColor': colors['background']}, children=[
        html.H1(
            children='My Finance Overview',
            style={
                'textAlign': 'center',
                'color': colors['text']
            }
        ),

        html.Div(children='A Summary of all my bank status', style={
            'textAlign': 'center',
            'color': colors['text']
        }),

        dcc.Graph(
            id='1',
            figure={
                'data': [{'type': 'indicator', 'value': curent_balance, 'title': 'Current Balance'}],
                'layout': general_chart_layout,
            }
        )
    ])

    if __name__ == '__main__':
        app.run_server(debug=True)

visualize()
```

In the end, I made several charts which shows:
-	**My current total balance**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Finally, we can answer the question at the beginning: how much money do I have right now?

-	**My current balance by banks** (e.g. RBC, CIBC, TD…)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;I know I have the money above, but how they distributed across all the banks I’m with? Furthermore, we can make decisions like: If I know bank B have a better deal for an investment, then should I consider move some money from bank A to bank B?

-	**My current balance by banks by accounts** (e.g. RBC chequing, RBC high interest, RBC Mastercard…)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Do I need to pay for my credit card balance right now? Should I move some money from my day to day chequing account to my high interest chequing account?

-	**Fixed monthly recurring fee** (e.g. condo fee, parking, car insurance, mobile bill, internet bill…)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Some fixed expenses to keep in mind you need to pay for every month

-	**Variable monthly recurring fee** (e.g. petro, gas, hydro…)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Some variable expense to pay for each month, and you can track the changing on these fees over time and get a better understanding about what change on your lifestyle caused the changes on these payments?

-	**Balance over month** 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Answer the question directly: I’m in red or green for this month? Or, should I buy this thing if I want to save up to $xxx for this month?

-	**Balance over month by income and expenses**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Answer a further question: I’m in red/green, because I got less/more income, or more/less expenses?

-	**Expenses by category** (e.g. online shopping, grocery, food delivery, education, travel…)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;What did you spend money on? Interestingly, I found since COVID started and I started to work from home, my grocery shopping expense got doubled at least compared with prior COVID months. 

Currently, I tend to run this workflow once a week, but actually you can run whenever you like. Now the charts are mostly focusing on monitoring spending since that’s the thing I want to start with, while I’m planning to add more sections for investment analysis as well when I got more bandwidth. 
