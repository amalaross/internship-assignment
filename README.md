# internship-assignment

import tkinter as tk
from tkinter import messagebox, ttk
import sqlite3
import feedparser
from tkinter import font as tkfont  # To modify fonts

# Predefined RSS Feeds
feeds = [
    "http://rss.cnn.com/rss/cnn_topstories.rss",
    "http://qz.com/feed",
    "http://feeds.foxnews.com/foxnews/politics",
    "http://feeds.reuters.com/reuters/businessNews",
    "http://feeds.feedburner.com/NewshourWorld",
    "https://feeds.bbci.co.uk/news/world/asia/india/rss.xml"
]

# Function to categorize articles based on their content
def categorize_article(summary):
    terrorism_keywords = ['terrorism','jail','Arrests','protest','activist','assault','fake','kill','killed','kidnapping','dead','political unrest','law','fraud','riot','justice','politics','violence','fear','lies','defamation','trap','criminal']
    positive_keywords = ['success', 'achievement','top','high','welcomed','positive','award','win','record-breaking', 'uplifting','victory','hopes','won','winner','champion','successfulness','outcome']
    disaster_keywords = ['earthquake','plane crash','flood', 'hurricane', 'natural disaster','disaster','tsunami','holocaust','lockdown',' trouble','holocaust','injuries','heavy rain']
    

    summary_lower = summary.lower()
    if any(keyword in summary_lower for keyword in terrorism_keywords):
        return 'Terrorism / protest / political unrest / riot'
    elif any(keyword in summary_lower for keyword in positive_keywords):
        return 'Positive/Uplifting'
    elif any(keyword in summary_lower for keyword in disaster_keywords):
        return 'Natural Disasters'
    else:
        return 'Others'

# Fetch articles from RSS feeds and categorize them
def fetch_news():
    articles = []
    for feed_url in feeds:
        feed = feedparser.parse(feed_url)
        for entry in feed.entries:
            title = entry.get('title', 'No title')
            link = entry.get('link', 'No link')
            summary = entry.get('summary', 'No summary available')
            published = entry.get('published', 'No publish date')
            category = categorize_article(summary)
            articles.append((title, link, summary, published, category))
    return articles

# Store the fetched articles in the SQLite database
def store_articles(articles):
    conn = sqlite3.connect('news.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS news (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT,
            link TEXT,
            summary TEXT,
            published DATE,
            category TEXT
        )
    ''')

    cursor.executemany('''
        INSERT INTO news (title, link, summary, published, category) 
        VALUES (?, ?, ?, ?, ?)
    ''', articles)
    
    conn.commit()
    conn.close()

# Function to fetch, categorize, and store the news
def fetch_and_store_news():
    articles = fetch_news()
    store_articles(articles)
    messagebox.showinfo("Info", "News articles fetched and stored successfully.")

# Function to display articles based on selected category and show the count
def display_articles(category):
    conn = sqlite3.connect('news.db')
    cursor = conn.cursor()
    
    if category == "All":
        cursor.execute("SELECT * FROM news ORDER BY id DESC")
    else:
        cursor.execute("SELECT * FROM news WHERE category=?", (category,))
    
    articles = cursor.fetchall()
    articles_text.delete(1.0, tk.END)  # Clear the text widget
    
    total_articles = len(articles)
    if total_articles > 0:
        articles_text.insert(tk.END, f"Total Articles in {category}: {total_articles}\n\n", 'bold')
    else:
        articles_text.insert(tk.END, f"No articles found in {category} category.\n\n", 'bold')
    
    for article in articles:
        articles_text.insert(tk.END, f"Title: {article[1]}\n", 'title')
        articles_text.insert(tk.END, f"Link: {article[2]}\n", 'link')
        articles_text.insert(tk.END, f"Summary: {article[3]}\n",'small')
        articles_text.insert(tk.END, f"Published: {article[4]}\n", 'small')
        articles_text.insert(tk.END, f"Category: {article[5]}\n\n", 'small')
        articles_text.insert(tk.END, "-" * 80 + "\n", 'separator')
    
    conn.close()

# Create the Tkinter window
root = tk.Tk()
root.title("News Article Categorizer")

# Create buttons to fetch news and display by category
fetch_button = tk.Button(root, text="Fetch and Store News", command=fetch_and_store_news)
fetch_button.grid(row=0, column=0, padx=10, pady=10)

category_label = tk.Label(root, text="Select Category:")
category_label.grid(row=1, column=0, padx=10, pady=10)

category_options = ["All", "Terrorism / protest / political unrest / riot", "Positive/Uplifting", "Natural Disasters", "Others"]
category_var = tk.StringVar(value="All")

category_menu = ttk.Combobox(root, textvariable=category_var, values=category_options, state="readonly")
category_menu.grid(row=1, column=1, padx=10, pady=10)

show_button = tk.Button(root, text="Show Articles", command=lambda: display_articles(category_var.get()))
show_button.grid(row=1, column=2, padx=10, pady=10)

# Frame to contain the text area and scrollbar
frame = tk.Frame(root)
frame.grid(row=2, column=0, columnspan=3, padx=10, pady=10)

# Text area to display articles
articles_text = tk.Text(frame, wrap=tk.WORD, width=150, height=35)
articles_text.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

# Scrollbar for the text widget
scrollbar = tk.Scrollbar(frame)
scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

# Configure the scrollbar to work with the text widget
articles_text.config(yscrollcommand=scrollbar.set)
scrollbar.config(command=articles_text.yview)

# Font Styles
articles_text.tag_configure('bold', font=('Helvetica', 14, 'bold'))
articles_text.tag_configure('title', font=('Times new roman', 12, 'bold'), foreground="blue")
articles_text.tag_configure('link', font=('Helvetica', 10, 'underline'), foreground="green")
articles_text.tag_configure('small', font=('Helvetica', 10))
articles_text.tag_configure('separator', foreground="gray")

# Start the Tkinter event loop
root.mainloop()
