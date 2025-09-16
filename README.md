# Ubuntu_Requests
import requests
import os
import hashlib
from urllib.parse import urlparse
from pathlib import Path

def create_image_fetcher():
    """
    Create an Ubuntu-inspired image fetcher that respects community and sharing principles
    """
    class UbuntuImageFetcher:
        def __init__(self):
            self.download_dir = "Fetched_Images"
            self.downloaded_hashes = set()
            self.load_existing_hashes()
            
        def load_existing_hashes(self):
            """Load hashes of already downloaded images to prevent duplicates"""
            if not os.path.exists(self.download_dir):
                return
                
            for file in os.listdir(self.download_dir):
                file_path = os.path.join(self.download_dir, file)
                if os.path.isfile(file_path):
                    try:
                        with open(file_path, 'rb') as f:
                            file_hash = hashlib.md5(f.read()).hexdigest()
                            self.downloaded_hashes.add(file_hash)
                    except:
                        continue
        
        def is_safe_url(self, url):
            """Check if URL appears to be safe for downloading"""
            parsed = urlparse(url)
            # Basic safety checks
            if not parsed.scheme in ('http', 'https'):
                return False, "URL must use HTTP or HTTPS protocol"
                
            # You could add more checks here for known malicious domains
            
            return True, "URL appears safe"
        
        def is_image_content(self, response):
            """Check if the response content appears to be an image"""
            content_type = response.headers.get('Content-Type', '').lower()
            image_types = ('image/jpeg', 'image/png', 'image/gif', 'image/webp', 
                          'image/bmp', 'image/svg+xml', 'image/tiff')
            
            if any(img_type in content_type for img_type in image_types):
                return True
                
            # Check magic numbers (file signatures) as a fallback
            magic_numbers = {
                b'\xFF\xD8\xFF': 'jpg',
                b'\x89PNG\r\n\x1a\n': 'png',
                b'GIF87a': 'gif',
                b'GIF89a': 'gif',
                b'RIFF....WEBP': 'webp',
                b'BM': 'bmp',
                b'MM\x00\x2a': 'tiff',
                b'II\x2a\x00': 'tiff'
            }
            
            content_start = response.content[:20]  # Check first 20 bytes
            for magic, ext in magic_numbers.items():
                if content_start.startswith(magic):
                    return True
                    
            return False
        
        def get_filename(self, url, response):
            """Extract filename from URL or generate one based on content"""
            parsed = urlparse(url)
            path = parsed.path
            filename = os.path.basename(path)
            
            # If no filename in URL, generate one based on content hash
            if not filename or '.' not in filename:
                content_hash = hashlib.md5(response.content).hexdigest()
                
                # Try to determine extension from Content-Type
                content_type = response.headers.get('Content-Type', '')
                if 'jpeg' in content_type or 'jpg' in content_type:
                    ext = 'jpg'
                elif 'png' in content_type:
                    ext = 'png'
                elif 'gif' in content_type:
                    ext = 'gif'
                else:
                    # Default to jpg if unknown
                    ext = 'jpg'
                    
                filename = f"{content_hash}.{ext}"
            
            return filename
        
        def is_duplicate(self, content):
            """Check if we've already downloaded this image"""
            content_hash = hashlib.md5(content).hexdigest()
            return content_hash in self.downloaded_hashes
        
        def fetch_image(self, url):
            """Fetch an image from a URL with proper error handling"""
            try:
                # Check URL safety first
                is_safe, message = self.is_safe_url(url)
                if not is_safe:
                    return False, message
                
                # Make the request with appropriate headers and timeout
                headers = {
                    'User-Agent': 'UbuntuImageFetcher/1.0 (Community Image Collection Tool)'
                }
                
                response = requests.get(url, headers=headers, timeout=15)
                response.raise_for_status()
                
                # Check if content is actually an image
                if not self.is_image_content(response):
                    return False, "URL does not point to an image file"
                
                # Check for duplicates
                if self.is_duplicate(response.content):
                    return False, "Image already exists in collection"
                
                # Get filename and save
                filename = self.get_filename(url, response)
                filepath = os.path.join(self.download_dir, filename)
                
                # Ensure directory exists
                os.makedirs(self.download_dir, exist_ok=True)
                
                # Save the image
                with open(filepath, 'wb') as f:
                    f.write(response.content)
                
                # Add to our hash set to prevent future duplicates
                content_hash = hashlib.md5(response.content).hexdigest()
                self.downloaded_hashes.add(content_hash)
                
                return True, f"Successfully fetched: {filename}"
                
            except requests.exceptions.RequestException as e:
                return False, f"Connection error: {e}"
            except Exception as e:
                return False, f"An error occurred: {e}"
        
        def fetch_multiple_images(self, urls):
            """Fetch multiple images from a list of URLs"""
            results = []
            for url in urls:
                url = url.strip()
                if url:  # Skip empty strings
                    success, message = self.fetch_image(url)
                    results.append((url, success, message))
            return results
    
    return UbuntuImageFetcher()

def main():
    print("Welcome to the Ubuntu Image Fetcher")
    print("A tool for mindfully collecting images from the web")
    print("""
Ubuntu Principles:
- Community: Connecting to the wider web community
- Respect: Handling resources and errors gracefully
- Sharing: Organizing images for later sharing
- Practicality: Creating a tool that serves a real need
""")
    
    # Create our image fetcher
    fetcher = create_image_fetcher()
    
    # Get URLs from user
    url_input = input("\nPlease enter image URLs (separated by commas): ")
    urls = [url.strip() for url in url_input.split(',') if url.strip()]
    
    if not urls:
        print("No URLs provided. Exiting.")
        return
    
    print(f"\nAttempting to fetch {len(urls)} image(s)...")
    
    # Fetch images
    results = fetcher.fetch_multiple_images(urls)
    
    # Print results
    success_count = 0
    for url, success, message in results:
        status = "✓" if success else "✗"
        print(f"{status} {url}: {message}")
        if success:
            success_count += 1
    
    # Final message
    print(f"\nFetched {success_count} of {len(urls)} images successfully.")
    print("Connection strengthened. Community enriched.")

if __name__ == "__main__":
    main()
