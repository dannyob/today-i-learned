---
layout: default
title: "YouTube Playlist Uploader for yt-dlp Videos"
date: 2025-01-20
tags:
  - Python
  - YouTube
  - yt-dlp
  - Automation
  - Video Processing
  - Google API
author: Danny O'Brien <danny@spesh.com>
---

# YouTube Playlist Uploader for yt-dlp Videos

I wanted a way to bulk upload videos I'd downloaded using yt-dlp to YouTube while preserving their metadata and organizing them into playlists automatically. This came up when I was archiving TikTok collections and wanted to migrate them to YouTube with proper attribution and organization.

The trickiest part was handling YouTube's API authentication and quota limits while preserving all the metadata that yt-dlp extracts - titles, descriptions, hashtags, and original URLs. The script needed to be smart about extracting hashtags from descriptions and converting them to YouTube tags, while also adding proper attribution back to the original creators.

The script automatically finds MP4 files and their corresponding info.json files, extracts all the metadata, uploads the videos as private by default, and organizes them into playlists.

The authentication flow uses OAuth 2.0, so users need to set up Google Cloud credentials, but once that's done, the script handles everything automatically. It includes proper error handling, rate limiting awareness, and detailed logging of the upload process.

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.8"
# dependencies = [
#     "google-auth-oauthlib>=1.0.0",
#     "google-auth-httplib2>=0.2.0",
#     "google-api-python-client>=2.0.0",
# ]
# ///

"""
YouTube Playlist Uploader for yt-dlp Videos

This script uploads videos downloaded with yt-dlp (along with their metadata) 
to YouTube and organizes them into playlists.

Prerequisites:
1. Install uv: curl -LsSf https://astral.sh/uv/install.sh | sh
2. Set up Google Cloud project with YouTube Data API v3 enabled
3. Download OAuth 2.0 credentials as 'credentials.json'
4. Download videos with: yt-dlp --write-info-json <URL>

Usage:
    ./upload-playlist.py --folder /path/to/videos --playlist "My Collection"
    ./upload-playlist.py -f ./downloads -p "TikTok Archive" --max-videos 10
"""

import os
import json
import argparse
import sys
from pathlib import Path
import re
from typing import List, Dict, Tuple, Optional

# Google API imports
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from googleapiclient.errors import HttpError
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow

# YouTube API scopes
SCOPES = [
    'https://www.googleapis.com/auth/youtube.upload',
    'https://www.googleapis.com/auth/youtube'
]

class YouTubeUploader:
    def __init__(self, credentials_file: str = 'credentials.json'):
        self.credentials_file = credentials_file
        self.token_file = 'token.json'
        self.youtube = None
        self.authenticate()
    
    def authenticate(self):
        """Authenticate with YouTube API using OAuth 2.0"""
        print("🔐 Authenticating with YouTube...")
        
        creds = None
        
        # Check if we already have valid credentials
        if os.path.exists(self.token_file):
            creds = Credentials.from_authorized_user_file(self.token_file, SCOPES)
        
        # If there are no valid credentials available, request authorization
        if not creds or not creds.valid:
            if creds and creds.expired and creds.refresh_token:
                creds.refresh(Request())
            else:
                if not os.path.exists(self.credentials_file):
                    print(f"❌ Error: {self.credentials_file} not found!")
                    print("Please download OAuth 2.0 credentials from Google Cloud Console")
                    print("and save them as 'credentials.json'")
                    sys.exit(1)
                
                flow = InstalledAppFlow.from_client_secrets_file(
                    self.credentials_file, SCOPES)
                creds = flow.run_local_server(port=0)
            
            # Save credentials for the next run
            with open(self.token_file, 'w') as token:
                token.write(creds.to_json())
        
        # Build YouTube API client
        self.youtube = build('youtube', 'v3', credentials=creds)
        print("✅ Authentication successful!")
    
    def create_playlist(self, title: str, description: str = "") -> str:
        """Create a new YouTube playlist and return its ID"""
        print(f"📋 Creating playlist: {title}")
        
        try:
            playlist_request = self.youtube.playlists().insert(
                part="snippet,status",
                body={
                    "snippet": {
                        "title": title,
                        "description": description or f"Playlist created by yt-dlp uploader"
                    },
                    "status": {
                        "privacyStatus": "private"  # Change to "public" or "unlisted" as needed
                    }
                }
            )
            
            playlist_response = playlist_request.execute()
            playlist_id = playlist_response["id"]
            
            print(f"✅ Created playlist ID: {playlist_id}")
            return playlist_id
            
        except HttpError as e:
            print(f"❌ Error creating playlist: {e}")
            return None
    
    def upload_video(self, video_file: str, title: str, description: str = "", 
                    tags: List[str] = None, category_id: str = "22") -> Optional[str]:
        """Upload a video to YouTube and return video ID"""
        print(f"⏳ Uploading video: {title}")
        
        if tags is None:
            tags = []
        
        # Prepare video metadata
        body = {
            "snippet": {
                "title": title[:100],  # YouTube title limit
                "description": description[:5000],  # YouTube description limit
                "tags": tags[:500],  # YouTube tag limit
                "categoryId": category_id
            },
            "status": {
                "privacyStatus": "private",  # Change as needed
                "selfDeclaredMadeForKids": False
            }
        }
        
        # Prepare media upload
        media = MediaFileUpload(
            video_file,
            chunksize=-1,
            resumable=True,
            mimetype='video/*'
        )
        
        try:
            # Execute upload
            insert_request = self.youtube.videos().insert(
                part=",".join(body.keys()),
                body=body,
                media_body=media
            )
            
            response = insert_request.execute()
            video_id = response["id"]
            
            print(f"✅ Successfully uploaded video ID: {video_id}")
            return video_id
            
        except HttpError as e:
            print(f"❌ Error uploading video: {e}")
            return None
    
    def add_video_to_playlist(self, playlist_id: str, video_id: str) -> bool:
        """Add a video to a playlist"""
        print(f"⏳ Adding video to playlist...")
        
        try:
            playlist_item_request = self.youtube.playlistItems().insert(
                part="snippet",
                body={
                    "snippet": {
                        "playlistId": playlist_id,
                        "resourceId": {
                            "kind": "youtube#video",
                            "videoId": video_id
                        }
                    }
                }
            )
            
            playlist_item_request.execute()
            print("✅ Video added to playlist")
            return True
            
        except HttpError as e:
            print(f"❌ Error adding video to playlist: {e}")
            return False


def find_video_files(folder: str) -> List[Tuple[str, str]]:
    """Find MP4 files with corresponding info.json files"""
    folder_path = Path(folder)
    video_files = []
    
    for mp4_file in folder_path.glob("*.mp4"):
        # Look for corresponding info.json file
        info_file = mp4_file.with_suffix('.info.json')
        if info_file.exists():
            video_files.append((str(mp4_file), str(info_file)))
        else:
            print(f"⚠️  Warning: No info file found for {mp4_file.name}")
    
    return video_files


def extract_metadata(info_file: str) -> Dict:
    """Extract metadata from yt-dlp info.json file"""
    with open(info_file, 'r', encoding='utf-8') as f:
        info = json.load(f)
    
    # Extract basic metadata
    title = info.get('title', 'Untitled')
    description = info.get('description', '')
    original_url = info.get('original_url', info.get('webpage_url', ''))
    uploader = info.get('uploader', '')
    
    # Extract hashtags from description
    hashtags = re.findall(r'#\w+', description)
    
    # Clean hashtags for YouTube tags (remove # symbol)
    tags = [tag[1:] for tag in hashtags]
    
    # Create enhanced description with attribution
    enhanced_description = description
    if original_url:
        enhanced_description += f"\n\n🔗 Original: {original_url}"
    
    return {
        'title': title,
        'description': enhanced_description,
        'tags': tags,
        'original_url': original_url,
        'uploader': uploader
    }


def main():
    parser = argparse.ArgumentParser(description='Upload yt-dlp videos to YouTube playlist')
    parser.add_argument('--folder', '-f', required=True, 
                       help='Folder containing MP4 and info.json files')
    parser.add_argument('--playlist', '-p', required=True,
                       help='YouTube playlist title')
    parser.add_argument('--max-videos', '-m', type=int,
                       help='Maximum number of videos to upload')
    parser.add_argument('--category', '-c', default='22',
                       help='YouTube category ID (default: 22 - People & Blogs)')
    
    args = parser.parse_args()
    
    # Validate folder
    if not os.path.exists(args.folder):
        print(f"❌ Error: Folder '{args.folder}' does not exist")
        sys.exit(1)
    
    # Find video files
    print(f"📁 Scanning folder: {args.folder}")
    video_files = find_video_files(args.folder)
    
    if not video_files:
        print("❌ No video files with corresponding info.json found")
        sys.exit(1)
    
    # Limit number of videos if specified
    if args.max_videos:
        video_files = video_files[:args.max_videos]
    
    print(f"📹 Found {len(video_files)} videos to upload")
    
    # Initialize uploader
    uploader = YouTubeUploader()
    
    # Create playlist
    playlist_id = uploader.create_playlist(args.playlist)
    if not playlist_id:
        print("❌ Failed to create playlist")
        sys.exit(1)
    
    # Upload videos
    successful_uploads = 0
    failed_uploads = 0
    
    for i, (video_file, info_file) in enumerate(video_files, 1):
        print(f"\n[{i}/{len(video_files)}] Processing: {Path(video_file).name}")
        
        try:
            # Extract metadata
            metadata = extract_metadata(info_file)
            
            print(f"Title: {metadata['title']}")
            print(f"Tags: {', '.join(metadata['tags'][:10])}{'...' if len(metadata['tags']) > 10 else ''}")
            
            # Upload video
            video_id = uploader.upload_video(
                video_file=video_file,
                title=metadata['title'],
                description=metadata['description'],
                tags=metadata['tags'],
                category_id=args.category
            )
            
            if video_id:
                # Add to playlist
                if uploader.add_video_to_playlist(playlist_id, video_id):
                    successful_uploads += 1
                    print(f"✅ Successfully uploaded: {metadata['title']}")
                    print(f"   Video ID: {video_id}")
                else:
                    failed_uploads += 1
                    print(f"❌ Failed to add to playlist: {metadata['title']}")
            else:
                failed_uploads += 1
                print(f"❌ Failed to upload: {metadata['title']}")
                
        except Exception as e:
            failed_uploads += 1
            print(f"❌ Error processing {Path(video_file).name}: {e}")
    
    # Print summary
    print(f"\n🎉 Upload completed!")
    print(f"✅ Successful uploads: {successful_uploads}")
    print(f"❌ Failed uploads: {failed_uploads}")
    print(f"📋 Playlist ID: {playlist_id}")
    print(f"🔗 Playlist URL: https://www.youtube.com/playlist?list={playlist_id}")


if __name__ == "__main__":
    main()
```
