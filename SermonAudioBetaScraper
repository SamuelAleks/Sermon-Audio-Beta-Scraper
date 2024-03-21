import urllib.parse
import os
import requests
import mutagen
from mutagen.id3 import ID3, TDRC
import tkinter as tk
from tkinter import filedialog
from datetime import datetime
import time
from selenium import webdriver
from selenium.webdriver.common.action_chains import ActionChains  # Import ActionChains
from selenium.webdriver.common.keys import Keys  # Import Keys
import re
import configparser

# Load the configuration file or create it if it doesn't exist
config = configparser.ConfigParser()
config_file = 'config.ini'

if not os.path.exists(config_file):
    # Create the 'Settings' section and set default values
    config['Settings'] = {'download_directory': ''}
    with open(config_file, 'w') as configfile:
        config.write(configfile)

config.read(config_file)

# Read the download directory from the configuration file
download_directory = config.get('Settings', 'download_directory')

# Create a tkinter window to show the directory selection dialog
root = tk.Tk()
root.withdraw()  # Hide the main window

# Ask the user to choose a directory for downloads or use the saved one
if not download_directory:
    download_directory = filedialog.askdirectory(title="Select a directory for downloads")
    # Save the selected directory to the configuration file
    if download_directory:
        config.set('Settings', 'download_directory', download_directory)
        with open(config_file, 'w') as configfile:
            config.write(configfile)

if download_directory:
    # Define the main URL

    #########################################
    # CHANGE URL TO MATCH THE ONE YOU NEED! #
    #########################################
    main_url = "https://beta.sermonaudio.com/speakers/<speaker_id>/sermons?sort=oldest"
    # URL Schema: https://beta/sermondaudio.com/speakers/<speaker_id>/sermons?sort=<option_modifiers>

    # Create a set to store the downloaded file names
    downloaded_files = set(os.listdir(download_directory))

    # Initialize the Firefox WebDriver
    from selenium.webdriver.firefox.service import Service
    from selenium.webdriver.firefox.options import Options

    options = Options()
    #options.headless = True  # Uncomment this line if you want to run Firefox in headless mode
    options.set_preference("browser.userAgent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Firefox/100.0")

    # Specify the path to the GeckoDriver executable
    geckodriver_path = "/usr/bin/geckodriver"  # Replace with the actual path to geckodriver

    driver = webdriver.Firefox(service=Service(geckodriver_path), options=options)
    driver.set_page_load_timeout(30)  # Increase the page load timeout to 30 seconds
    driver.implicitly_wait(30)

    # Send an HTTP GET request to the main URL
    driver.get(main_url)

    # Check if the request was successful
    # if "SermonAudio" in driver.title:
    if "" in driver.title:
        # Initialize variables for scrolling
        start_time = time.time()  # Record the start time
        timeout = 30  # Set a timeout for 10 seconds
        scroll_step = 10  # Set the scroll step size (adjust as needed)

        while True:
            page_height = driver.execute_script("return document.body.scrollHeight")
            scroll_height = page_height

            # Scroll down by sending "pagedown" key events using ActionChains in smaller steps
            for _ in range(page_height // scroll_step):
                action = ActionChains(driver)
                action.send_keys(Keys.PAGE_DOWN)
                action.perform()
                time.sleep(0.1)  # Sleep briefly between each scroll step

            scroll_height = driver.execute_script("return document.body.scrollHeight")

            # Check if the timeout has been reached
            if time.time() - start_time > timeout:
                print("Timeout reached. Exiting the scrolling loop.")
                break

            # Check if no new elements have loaded (page height remains the same)
            if page_height == scroll_height:
                break
        # Get the updated page content with dynamically loaded content
        updated_html = driver.page_source

        # Parse the updated page content with BeautifulSoup (you'll need to import BeautifulSoup as well)
        from bs4 import BeautifulSoup

        soup = BeautifulSoup(updated_html, 'html.parser')

        # Find the master div with class "mb-2 mt-2"
        master_div = soup.find('div', class_='mb-2 mt-2')

        # Find all first-layer child divs under the master div
        div_elements = master_div.find_all('div', recursive=False)
        print(f"Found {len(div_elements)} div elements.")
        
        for div in div_elements:
            # Find the links with "sermon" in their href attributes within each list item
            sermon_links = div.find_all('a', href=re.compile('.*sermon.*', re.IGNORECASE))

            # Find the link with the speaker information
            speaker_link = div.find('a', href=re.compile(r'/speakers/\d+'))

            # Check if there are any "sermon" links in this div and the speaker link
            if sermon_links and speaker_link:
                # Extract the relative URL from the sermon link
                relative_sermon_url = sermon_links[0]['href']

                # Extract the author name from the speaker link
                author = speaker_link.text.strip()

                # Construct the complete URL
                complete_sermon_url = urllib.parse.urljoin(main_url, relative_sermon_url)

                sermon_response = requests.get(complete_sermon_url)

                if sermon_response.status_code == 200:
                    sermon_soup = BeautifulSoup(sermon_response.text, 'html.parser')

                    # Find the text following the <h1> tag with class "text-2xl"
                    h1_tag = sermon_soup.find('h1', class_='text-2xl')
                    if h1_tag:
                        file_title = h1_tag.text.strip()
                    else:
                        file_title = "Untitled"  # Use a default title if not found

                    # Find the text following <td class="value" data-v-6076b015="">
                    date_created_tds = sermon_soup.find_all('td', class_='value', attrs={'data-v-6076b015': True})

                    # Check if there's at least one matching element
                    if date_created_tds:
                        date_created_text = date_created_tds[2].text.strip()  # Use the third element (index 2)

                        # Convert date format
                        try:
                            date_obj = datetime.strptime(date_created_text, '%b %d, %Y')
                            formatted_date = date_obj.strftime('%Y-%m-%d')
                        except ValueError as e:
                            print(f"Error converting date: {e}")
                            formatted_date = ""
                    else:
                        date_created_text = ""
                        formatted_date = ""

                    # Append author and date_created_text to the filename
                    file_name = f"{author} - {formatted_date} - {file_title}.mp3"

                    # Check if the file already exists in the download directory
                    if file_name not in downloaded_files:
                        # Construct the complete path to save the file
                        file_path = os.path.join(download_directory, file_name)

                        # Download the mp3 file
                        mp3_link = sermon_soup.find('a', href=re.compile('.*\.mp3'))
                        if mp3_link:
                            mp3_url = mp3_link['href']

                            if not mp3_url.startswith('http'):
                                mp3_url = urllib.parse.urljoin(complete_sermon_url, mp3_url)

                            mp3_response = requests.get(mp3_url)

                            if mp3_response.status_code == 200:
                                # Save the mp3 file to the selected directory
                                with open(file_path, 'wb') as mp3_file:
                                    mp3_file.write(mp3_response.content)

                                # Add date_created_text to the "date created" metadata
                                audio = mutagen.File(file_path, easy=True)
                                if 'date' not in audio:
                                    audio['date'] = formatted_date

                                # Add author to the "Author" metadata
                                audio['author'] = author

                                audio.save()

                                print(f"MP3 file '{file_name}' downloaded successfully to '{download_directory}'.")
                            else:
                                print(f"Failed to download MP3 file from '{mp3_url}'.")
                        else:
                            print("No .mp3 link found on the sermon page.")
                    else:
                        print(f"Skipping '{file_name}' as it already exists in the download directory.")
                else:
                    print("Failed to fetch sermon page.")
    else:
        print("Failed to fetch main page.")
else:
    print("No directory selected. Download canceled.")
