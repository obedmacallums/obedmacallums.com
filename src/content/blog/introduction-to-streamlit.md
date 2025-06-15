---
author: Obed Macallums
pubDatetime: 2025-06-15T16:30:00Z
modDatetime: 2025-06-15T16:30:00Z
title: "Introduction to Streamlit"
slug: introduction-to-streamlit
featured: false
draft: false
tags:
  - python
  - streamlit
  - web-development
  - data-science
description: "An easy-to-follow introduction to Streamlit for building interactive web applications in Python."
---

Streamlit is a revolutionary Python library that transforms the way data scientists and developers create web applications. This tutorial provides an easy-to-follow introduction to Streamlit, covering everything from installation to building your first interactive app. By the end of this post, you'll have a solid understanding of how to use Streamlit to turn your Python scripts into beautiful, shareable web applications.

## Table of contents

## What is Streamlit?

Streamlit is an open-source Python framework that makes it incredibly easy to create and share beautiful, custom web applications for machine learning and data science projects. It allows you to build interactive web apps with just a few lines of Python code, without requiring any knowledge of HTML, CSS, or JavaScript.

Some of the key features of Streamlit include:

- **Simple API** for creating interactive widgets and layouts
- **Real-time updates** as you edit your Python script
- **Built-in support** for popular data science libraries like Pandas, Matplotlib, and Plotly
- **Easy deployment** to share your apps with others
- **Caching mechanisms** for improved performance with large datasets

## Installing Streamlit

Installing Streamlit is straightforward with pip:

```bash
pip install streamlit
```

To verify your installation and see a demo app, run:

```bash
streamlit hello
```

This will open a browser window with Streamlit's demo application, showcasing various features and capabilities.

## Your First Streamlit App

Let's create a simple "Hello World" Streamlit app. Create a new Python file called `app.py`:

```python
import streamlit as st

# Set page title
st.title("My First Streamlit App")

# Add some text
st.write("Hello, World!")

# Add a slider
age = st.slider("How old are you?", 0, 130, 25)
st.write("You are", age, "years old")

# Add a text input
name = st.text_input("What's your name?")
if name:
    st.write(f"Hello, {name}!")
```

To run your app, save the file and execute:

```bash
streamlit run app.py
```

Your browser will automatically open to display your interactive web application!

## Core Streamlit Components

Streamlit provides a variety of components for building interactive applications:

### Text and Data Display

```python
import streamlit as st
import pandas as pd

# Display text with different formats
st.title("Main Title")
st.header("Header")
st.subheader("Subheader")
st.text("Plain text")
st.markdown("**Bold text** and *italic text*")

# Display data
data = {"Name": ["Alice", "Bob", "Charlie"], "Age": [25, 30, 35]}
df = pd.DataFrame(data)
st.dataframe(df)  # Interactive dataframe
st.table(df)      # Static table
```

### Input Widgets

```python
# Various input widgets
text_input = st.text_input("Enter some text")
number = st.number_input("Pick a number", min_value=0, max_value=100)
date = st.date_input("Pick a date")
option = st.selectbox("Choose an option", ["Option 1", "Option 2", "Option 3"])
multi_select = st.multiselect("Choose multiple", ["A", "B", "C", "D"])
checkbox = st.checkbox("Check me out")
```

### File Uploads

```python
uploaded_file = st.file_uploader("Choose a CSV file", type="csv")
if uploaded_file is not None:
    df = pd.read_csv(uploaded_file)
    st.write(df)
```

## Building Interactive Data Applications

Streamlit shines when building data-driven applications. Here's an example of a simple data exploration app:

```python
import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt

st.title("Data Explorer")

# File upload
uploaded_file = st.file_uploader("Upload a CSV file", type="csv")

if uploaded_file is not None:
    # Read data
    df = pd.read_csv(uploaded_file)
    
    # Display basic info
    st.subheader("Dataset Overview")
    st.write(f"Shape: {df.shape}")
    st.write(df.head())
    
    # Column selection for visualization
    st.subheader("Data Visualization")
    columns = df.select_dtypes(include=['float64', 'int64']).columns.tolist()
    
    if len(columns) >= 2:
        x_axis = st.selectbox("Select X-axis", columns)
        y_axis = st.selectbox("Select Y-axis", columns)
        
        # Create plot
        fig, ax = plt.subplots()
        ax.scatter(df[x_axis], df[y_axis])
        ax.set_xlabel(x_axis)
        ax.set_ylabel(y_axis)
        st.pyplot(fig)
```

## Layout and Styling

Streamlit provides several options for organizing your app's layout:

### Columns

```python
col1, col2, col3 = st.columns(3)

with col1:
    st.header("Column 1")
    st.write("Content for column 1")

with col2:
    st.header("Column 2")
    st.write("Content for column 2")

with col3:
    st.header("Column 3")
    st.write("Content for column 3")
```

### Sidebar

```python
# Add widgets to sidebar
st.sidebar.header("Settings")
filter_option = st.sidebar.selectbox("Filter by", ["All", "Category A", "Category B"])
min_value = st.sidebar.slider("Minimum value", 0, 100, 10)
```

### Containers and Tabs

```python
# Using containers
container = st.container()
with container:
    st.write("This is inside a container")

# Using tabs
tab1, tab2, tab3 = st.tabs(["Tab 1", "Tab 2", "Tab 3"])

with tab1:
    st.write("Content for tab 1")

with tab2:
    st.write("Content for tab 2")

with tab3:
    st.write("Content for tab 3")
```

## Caching for Performance

When working with large datasets or expensive computations, Streamlit's caching can significantly improve performance:

```python
@st.cache_data
def load_data(file_path):
    """Load data with caching to improve performance"""
    return pd.read_csv(file_path)

@st.cache_data
def expensive_computation(data):
    """Cache expensive computations"""
    # Some heavy computation
    return processed_data

# Usage
df = load_data("large_dataset.csv")
result = expensive_computation(df)
```

## Deployment Options

Streamlit offers several ways to deploy your applications:

### Streamlit Community Cloud

1. Push your code to a GitHub repository
2. Visit [share.streamlit.io](https://share.streamlit.io)
3. Connect your GitHub account and select your repository
4. Your app will be deployed automatically!

### Local Deployment

For local sharing, simply run:

```bash
streamlit run app.py --server.port 8080
```

### Docker Deployment

Create a `Dockerfile`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8501

CMD ["streamlit", "run", "app.py"]
```

## Best Practices

Here are some best practices for building Streamlit applications:

- **Use caching** for expensive operations and data loading
- **Organize your code** into functions and modules for better maintainability
- **Add error handling** to make your app more robust
- **Use session state** to maintain state across reruns when needed
- **Test your app** with different inputs and edge cases
- **Add documentation** and help text for user guidance

```python
# Example of session state usage
if 'counter' not in st.session_state:
    st.session_state.counter = 0

if st.button('Increment'):
    st.session_state.counter += 1

st.write(f'Counter: {st.session_state.counter}')
```

## Conclusion

Streamlit is a game-changing tool that democratizes web application development for data scientists and Python developers. With its intuitive API and powerful features, you can quickly transform your data analysis scripts into interactive, shareable web applications. Understanding the fundamentals of widgets, layouts, caching, and deployment will enable you to build sophisticated data applications with minimal effort.

For next steps, consider exploring the [Streamlit documentation](https://docs.streamlit.io/) to discover advanced features like custom components and multi-page apps. You could also try building real-world applications such as dashboards for business metrics, machine learning model demos, or data exploration tools for your specific domain.
