---
layout: default
title: "Scraping photos from a Pingboard directory"
date: 2025-01-15
tags:
  - Python
  - Playwright
  - Web Scraping
  - Automation
  - Employee Directory
author: Danny O'Brien <danny@spesh.com>
---

# Scraping employee photos from Pingboard directory

I wanted to download all the photos from a [Pingboard](https://pingboard.com/) directory for a project. Pingboard is an employee directory service that displays staff photos and information, but there's no built-in way to bulk export the photos.

This script uses Playwright to connect to an existing browser session with Chrome Remote Debugging (avoiding login complications), scrapes the employee directory page to extract photo URLs and employee information, then downloads all the photos with descriptive filenames.

The trickiest part was handling Pingboard's dynamic loading - the directory uses JavaScript to render the employee grid, so I had to wait for the content to load and execute JavaScript to extract the data. The script also handles pagination by automatically clicking "More" buttons to load additional employees.

I used Python with async/await throughout to make the downloads concurrent and faster. The script saves a JSON log of all the download attempts, making it easy to retry failed downloads or process the data later.

To use it, start Chrome with remote debugging enabled (`chrome --remote-debugging-port=9222`), log into Pingboard in that browser, then run the script. 

This was written with Claude (Opus 4 and a bit of Sonnet cleanup).


```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.8"
# dependencies = [
#     "playwright>=1.40.0",
#     "aiohttp>=3.8.0",
#     "aiofiles>=23.0.0",
#     "requests>=2.28.0",
# ]
# ///

import asyncio
import os
import json
import sys
from playwright.async_api import async_playwright
import aiohttp
import aiofiles
from datetime import datetime

def detect_image_format(content_type, url, content):
    """Detect image format from content-type, URL, or content"""
    if content_type:
        if 'svg' in content_type.lower():
            return 'svg'
        elif 'png' in content_type.lower():
            return 'png'
        elif 'gif' in content_type.lower():
            return 'gif'
        elif 'webp' in content_type.lower():
            return 'webp'
        elif 'jpeg' in content_type.lower() or 'jpg' in content_type.lower():
            return 'jpg'
    
    # Check URL extension
    url_lower = url.lower()
    if url_lower.endswith('.svg'):
        return 'svg'
    elif url_lower.endswith('.png'):
        return 'png'
    elif url_lower.endswith('.gif'):
        return 'gif'
    elif url_lower.endswith('.webp'):
        return 'webp'
    elif url_lower.endswith('.jpg') or url_lower.endswith('.jpeg'):
        return 'jpg'
    
    # Check content headers (magic bytes)
    if content:
        if content.startswith(b'<svg') or content.startswith(b'<?xml'):
            return 'svg'
        elif content.startswith(b'\x89PNG'):
            return 'png'
        elif content.startswith(b'GIF8'):
            return 'gif'
        elif content.startswith(b'RIFF') and b'WEBP' in content[:12]:
            return 'webp'
        elif content.startswith(b'\xff\xd8\xff'):
            return 'jpg'
    
    # Default to jpg if we can't detect
    return 'jpg'

async def download_image(session, url, base_filepath):
    """Download an image from URL to filepath with auto-detected extension"""
    try:
        async with session.get(url) as response:
            if response.status == 200:
                content = await response.read()
                content_type = response.headers.get('content-type', '')
                
                # Detect the actual image format
                image_format = detect_image_format(content_type, url, content)
                
                # Update filepath with correct extension
                base_name = os.path.splitext(base_filepath)[0]
                filepath = f"{base_name}.{image_format}"
                
                async with aiofiles.open(filepath, 'wb') as f:
                    await f.write(content)
                
                return True, image_format, os.path.basename(filepath)
    except Exception as e:
        print(f"Error downloading {url}: {e}")
    return False, None, None

def build_directory_url(domain):
    """Build the directory URL from a domain, handling various input formats"""
    # Remove protocol if present
    if domain.startswith('http://') or domain.startswith('https://'):
        domain = domain.split('://', 1)[1]
    
    # Remove trailing slash and /users/directory if present
    domain = domain.rstrip('/')
    if domain.endswith('/users/directory'):
        domain = domain[:-len('/users/directory')]
    elif domain.endswith('/users'):
        domain = domain[:-len('/users')]
    
    # Build full URL
    return f"https://{domain}/users/directory"

async def scrape_pingboard_photos(pingboard_domain):
    # Build the directory URL
    directory_url = build_directory_url(pingboard_domain)
    print(f"Target URL: {directory_url}")
    
    # Create output directory with timestamp and domain
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    domain_name = pingboard_domain.split('.')[0].replace('https://', '').replace('http://', '')
    output_dir = f"pingboard_photos_{domain_name}_{timestamp}"
    os.makedirs(output_dir, exist_ok=True)
    
    # Create a log file for the scraping session
    log_file = os.path.join(output_dir, "download_log.json")
    
    async with async_playwright() as p:
        # Connect to existing Chrome instance with remote debugging
        # First, start Chrome with: chrome --remote-debugging-port=9222
        
        print("Connecting to Chrome DevTools on port 9222...")
        print("Make sure Chrome is running with: chrome --remote-debugging-port=9222")
        
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9222")
        except Exception as e:
            print(f"Failed to connect to Chrome: {e}")
            print("\nTo start Chrome with debugging:")
            print("Windows: chrome.exe --remote-debugging-port=9222")
            print("Mac: /Applications/Google\\ Chrome.app/Contents/MacOS/Google\\ Chrome --remote-debugging-port=9222")
            print("Linux: google-chrome --remote-debugging-port=9222")
            return
        
        contexts = browser.contexts
        if not contexts:
            print("No browser contexts found.")
            return
            
        context = contexts[0]
        pages = context.pages
        page = pages[0] if pages else await context.new_page()
        
        # Navigate to directory
        current_url = page.url
        
        if directory_url not in current_url:
            print(f"Navigating to {directory_url}...")
            await page.goto(directory_url, wait_until="networkidle")
            await asyncio.sleep(3)
        
        print("Extracting user data from directory...")
        
        # Execute JavaScript to extract all user data at once
        users_data = await page.evaluate("""
            () => {
                // Find the main grid container
                const userGrid = document.querySelector('ul.grid') || 
                                document.querySelector('ul[class*="grid"]');
                
                if (!userGrid) {
                    return { error: "Could not find user grid" };
                }
                
                // Get all user list items
                const userListItems = userGrid.querySelectorAll('li');
                
                // Extract data from each user
                const users = Array.from(userListItems).map((li, index) => {
                    const link = li.querySelector('a[href*="/users/"]');
                    const img = li.querySelector('img');
                    
                    if (!link || !img) return null;
                    
                    // Extract user ID from URL
                    const userId = link.href.split('/users/')[1];
                    
                    // Get name from alt text (cleaner than parsing fullText)
                    const name = img.alt || '';
                    
                    // Get full text for job title extraction
                    const fullText = li.textContent.trim();
                    
                    // Try to extract job title by removing the name from fullText
                    let jobTitle = fullText;
                    if (name && fullText.startsWith(name)) {
                        jobTitle = fullText.substring(name.length).trim();
                    }
                    
                    return {
                        index,
                        userId,
                        profileUrl: link.href,
                        imageUrl: img.src,
                        name: name,
                        jobTitle: jobTitle,
                        fullText: fullText
                    };
                }).filter(user => user !== null);
                
                return {
                    totalFound: userListItems.length,
                    validUsers: users.length,
                    users: users
                };
            }
        """)
        
        if 'error' in users_data:
            print(f"Error: {users_data['error']}")
            await browser.close()
            return
        
        print(f"Found {users_data['validUsers']} valid users out of {users_data['totalFound']} total entries")
        
        # Prepare download summary
        download_summary = {
            "timestamp": timestamp,
            "pingboard_domain": pingboard_domain,
            "directory_url": directory_url,
            "total_users": users_data['validUsers'],
            "downloads": []
        }
        
        # Download images
        successful = 0
        failed = 0
        
        async with aiohttp.ClientSession() as session:
            for i, user in enumerate(users_data['users'], 1):
                name = user['name']
                if not name:
                    print(f"  ⚠ Skipping user {user['userId']} - no name found")
                    failed += 1
                    continue
                
                # Clean filename (without extension - will be added based on detected format)
                base_filename = name
                # Remove any invalid filename characters
                base_filename = "".join(c for c in base_filename if c.isalnum() or c in (' ', '-', '_')).strip()
                base_filepath = os.path.join(output_dir, base_filename)
                
                print(f"[{i}/{users_data['validUsers']}] Downloading: {name}")
                
                success, image_format, final_filename = await download_image(session, user['imageUrl'], base_filepath)
                
                download_info = {
                    "name": name,
                    "filename": final_filename,
                    "image_format": image_format,
                    "userId": user['userId'],
                    "profileUrl": user['profileUrl'],
                    "imageUrl": user['imageUrl'],
                    "jobTitle": user['jobTitle'],
                    "success": success
                }
                
                download_summary['downloads'].append(download_info)
                
                if success:
                    format_indicator = f" ({image_format.upper()})" if image_format != 'jpg' else ""
                    print(f"  ✓ Saved as {final_filename}{format_indicator}")
                    successful += 1
                else:
                    print(f"  ✗ Failed to download")
                    failed += 1
                
                # Small delay between downloads
                await asyncio.sleep(0.5)
        
        # Save download summary
        with open(log_file, 'w', encoding='utf-8') as f:
            json.dump(download_summary, f, indent=2, ensure_ascii=False)
        
        await browser.close()
        
        print(f"\n{'='*50}")
        print(f"Download complete!")
        print(f"✓ Successful: {successful}")
        print(f"✗ Failed: {failed}")
        print(f"📁 Photos saved in: {output_dir}")
        print(f"📄 Log file: {log_file}")
        print(f"{'='*50}")

# Additional utility function to process existing directory data
def process_directory_json(json_file, output_dir="processed_photos"):
    """
    Process a JSON file containing user data and download photos.
    Useful if you've already extracted the data and want to download later.
    """
    import json
    import requests
    
    os.makedirs(output_dir, exist_ok=True)
    
    with open(json_file, 'r', encoding='utf-8') as f:
        data = json.load(f)
    
    users = data.get('users', [])
    print(f"Processing {len(users)} users from JSON...")
    
    successful = 0
    failed = 0
    
    for i, user in enumerate(users, 1):
        name = user.get('altText', '') or user.get('name', '')
        if not name:
            print(f"Skipping user {i} - no name found")
            failed += 1
            continue
        
        base_filename = name
        base_filename = "".join(c for c in base_filename if c.isalnum() or c in (' ', '-', '_')).strip()
        
        print(f"[{i}/{len(users)}] Downloading: {name}")
        
        try:
            response = requests.get(user['imageUrl'], timeout=30)
            if response.status_code == 200:
                content = response.content
                content_type = response.headers.get('content-type', '')
                
                # Detect image format
                image_format = detect_image_format(content_type, user['imageUrl'], content)
                filename = f"{base_filename}.{image_format}"
                filepath = os.path.join(output_dir, filename)
                
                with open(filepath, 'wb') as f:
                    f.write(content)
                
                format_indicator = f" ({image_format.upper()})" if image_format != 'jpg' else ""
                print(f"  ✓ Saved as {filename}{format_indicator}")
                successful += 1
            else:
                print(f"  ✗ Failed with status {response.status_code}")
                failed += 1
        except Exception as e:
            print(f"  ✗ Error: {e}")
            failed += 1
    
    print(f"\nComplete! Successful: {successful}, Failed: {failed}")

def print_usage():
    """Print usage information"""
    print("Usage: python scrape_pingboard.py <pingboard_domain>")
    print("\nExamples:")
    print("  python scrape_pingboard.py yourcompany.pingboard.com")
    print("  python scrape_pingboard.py https://yourcompany.pingboard.com")
    print("  python scrape_pingboard.py yourcompany.pingboard.com/users/directory")
    print("\nThe script will automatically add /users/directory to the URL if needed.")

# Run the scraper
if __name__ == "__main__":
    if len(sys.argv) != 2:
        print_usage()
        sys.exit(1)
    
    pingboard_domain = sys.argv[1]
    
    # Validate that it looks like a pingboard domain
    if 'pingboard.com' not in pingboard_domain:
        print(f"Warning: '{pingboard_domain}' doesn't appear to be a Pingboard domain.")
        response = input("Continue anyway? (y/N): ")
        if response.lower() not in ['y', 'yes']:
            print("Aborted.")
            sys.exit(1)
    
    print(f"Starting scraper for: {pingboard_domain}")
    asyncio.run(scrape_pingboard_photos(pingboard_domain))
```
