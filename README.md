
---

# Video Transfer Solution: traq.ai to Azure Blob Storage

## Executive Summary

I have developed a robust Python-based solution to efficiently transfer call transcript videos from traq.ai (Google Cloud Storage) to Azure Blob Storage. This document outlines the process, requirements, implementation details, and provides the full source code for review and implementation.

## 1. Project Overview

### 1.1 Objective

To securely and efficiently transfer video files from traq.ai's Google Cloud Storage to our Azure Blob Storage, ensuring data integrity and transfer reliability.

### 1.2 Key Features

* Concurrent file transfers for improved efficiency
* Comprehensive error handling and logging
* Detailed transfer reporting
* Progress tracking for large-scale transfers
* Customizable file paths in Azure Blob Storage

## 2. Technical Requirements

### 2.1 Source (traq.ai)

* **Signed URLs** : A list of pre-signed URLs for each video, containing embedded access parameters.
* **URL Format Example** :

```
  https://storage.googleapis.com/app.traq365.com/b4cd19f0-1f06-4496-b03a-9a188c8f8307%2F338a3a7f-9013-494b-b003-e099f24fa93b%2Fvideo.mp4?GoogleAccessId=traq365storageaccount@traq365-web-app.iam.gserviceaccount.com&Expires=1729604826&Signature=YamVxm8WCLy%2F0zUHsdCabUl9p4Tw0R3dLl3si%2BwNVjJktIpaYq5kGcMUql7qyQ33HYdqBpxcgsrWKDVz1aKiGw24Ju3iwSMAoyey3GuW6oLeEn0a05DhF7ocBC%2FkghAwk5nJvHTqf6MEOXOr5ZNAZQTY%2BfnaXuRqgElt7cvHdCnw6%2B27ypwCa6NO39tDzLMXFLL1Nov15VOdkgvaryDVJHZTGbGt83F3nA%2Fft%2BPhwzf9lJm9A%2BQkFdZJmu1KYF7Yg5WCXFQzRntFymT7iBfmFO7K9VsMieMLVyzmZThQLm3sym3iCdGfJTaLs%2BdGsl4zNAvZ2xaKcECDBCbUya9j2Q%3D%3D
```

### 2.2 Destination (Azure Blob Storage)

* **Azure Connection String** : Contains account credentials.
  Format: `"DefaultEndpointsProtocol=https;AccountName=<your_account_name>;AccountKey=<your_account_key>;EndpointSuffix=core.windows.net"`
* **Container Name** : The name of the Azure Blob Storage container for storing videos.

### 2.3 Development Environment

* Python 3.6+
* Required packages: `azure-storage-blob`, `requests`, `tqdm`

## 3. Implementation Details

### 3.1 Core Components

* `TraqToAzureTransfer` class: Manages the entire transfer process
* Concurrent processing using `ThreadPoolExecutor`
* Error handling and logging mechanism
* Progress tracking with `tqdm`
* Customizable file paths in Azure

### 3.2 Key Methods

* `__init__`: Initializes Azure client and logging
* `extract_filename_from_url`: Generates unique filenames from traq.ai URLs
* `transfer_file`: Handles individual file transfers
* `bulk_transfer`: Manages the entire transfer process for multiple files

## 4. Source Code

```python
import requests
from azure.storage.blob import BlobServiceClient
import os
from datetime import datetime
import logging
from urllib.parse import urlparse, unquote
from concurrent.futures import ThreadPoolExecutor, as_completed
from tqdm import tqdm
import json

class TraqToAzureTransfer:
    def __init__(self, azure_connection_string, max_workers=5):
        """
        Initialize the transfer client with Azure credentials
      
        Args:
            azure_connection_string: Azure Storage connection string
            max_workers: Maximum number of concurrent transfers (default: 5)
        """
        self.max_workers = max_workers
      
        # Set up logging
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            filename=f'transfer_log_{datetime.now().strftime("%Y%m%d_%H%M%S")}.log'
        )
      
        try:
            # Initialize Azure Blob Storage client
            self.azure_client = BlobServiceClient.from_connection_string(
                azure_connection_string
            )
            logging.info("Successfully initialized Azure client")
          
        except Exception as e:
            logging.error(f"Error initializing Azure client: {str(e)}")
            raise

    def extract_filename_from_url(self, url):
        """Extract original filename from traq.ai URL"""
        # Parse the URL and get the path
        path = unquote(urlparse(url).path)
      
        # Extract the filename components
        components = path.split('/')
        if len(components) >= 4:
            # Use the GUIDs to create a unique filename
            guid1 = components[-3]
            guid2 = components[-2]
            base_name = f"{guid1}_{guid2}"
        else:
            base_name = components[-1]
      
        # Ensure .mp4 extension
        if not base_name.endswith('.mp4'):
            base_name = f"{base_name}.mp4"
      
        return base_name

    def transfer_file(self, source_url, azure_container_name, azure_blob_path=None):
        """
        Transfer a single file from traq.ai URL to Azure Blob Storage
      
        Args:
            source_url: Source URL from traq.ai
            azure_container_name: Destination container name in Azure
            azure_blob_path: Optional custom path for the file in Azure
      
        Returns:
            dict: Transfer result with status and details
        """
        try:
            # If no Azure path specified, extract from URL
            if azure_blob_path is None:
                azure_blob_path = self.extract_filename_from_url(source_url)
          
            # Get the destination container
            container_client = self.azure_client.get_container_client(
                azure_container_name
            )
          
            # Download from URL
            response = requests.get(source_url, stream=True)
            response.raise_for_status()
          
            # Get file size for progress tracking
            file_size = int(response.headers.get('content-length', 0))
          
            # Upload to Azure using streaming
            blob_client = container_client.upload_blob(
                name=azure_blob_path,
                data=response.raw,
                overwrite=True,
                length=file_size
            )
          
            logging.info(
                f"Successfully transferred {azure_blob_path} to Azure"
            )
          
            return {
                'status': 'success',
                'source_url': source_url,
                'destination_path': azure_blob_path,
                'size': file_size
            }
          
        except Exception as e:
            error_msg = str(e)
            logging.error(
                f"Error transferring file {source_url}: {error_msg}"
            )
            return {
                'status': 'failed',
                'source_url': source_url,
                'error': error_msg
            }

    def bulk_transfer(self, url_list, azure_container_name, 
                      custom_paths=None, save_report=True):
        """
        Transfer multiple files concurrently from traq.ai URLs to Azure
      
        Args:
            url_list: List of source URLs from traq.ai
            azure_container_name: Destination container name in Azure
            custom_paths: Optional dictionary mapping URLs to custom Azure paths
            save_report: Whether to save transfer report to file (default: True)
      
        Returns:
            dict: Summary of transfer results
        """
        if custom_paths is None:
            custom_paths = {}
      
        results = {
            'successful': [],
            'failed': [],
            'start_time': datetime.now().isoformat(),
            'total_files': len(url_list)
        }
      
        # Create progress bar
        pbar = tqdm(total=len(url_list), desc="Transferring files")
      
        # Use ThreadPoolExecutor for concurrent transfers
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            # Submit all transfer tasks
            future_to_url = {
                executor.submit(
                    self.transfer_file,
                    url,
                    azure_container_name,
                    custom_paths.get(url)
                ): url for url in url_list
            }
          
            # Process completed transfers
            for future in as_completed(future_to_url):
                result = future.result()
                if result['status'] == 'success':
                    results['successful'].append(result)
                else:
                    results['failed'].append(result)
                pbar.update(1)
      
        pbar.close()
      
        # Add summary statistics
        results['end_time'] = datetime.now().isoformat()
        results['summary'] = {
            'total_files': len(url_list),
            'successful_transfers': len(results['successful']),
            'failed_transfers': len(results['failed']),
            'success_rate': f"{(len(results['successful']) / len(url_list) * 100):.2f}%"
        }
      
        # Save detailed report if requested
        if save_report:
            report_name = f"transfer_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
            with open(report_name, 'w') as f:
                json.dump(results, f, indent=2)
            logging.info(f"Transfer report saved to {report_name}")
      
        return results

# Example usage
if __name__ == "__main__":
    # Replace with your Azure connection string
    AZURE_CONNECTION_STRING = "your-azure-connection-string"
  
    # Example list of traq.ai URLs
    TRAQ_URLS = [
        "https://storage.googleapis.com/app.traq365.com/.../video1.mp4?...",
        "https://storage.googleapis.com/app.traq365.com/.../video2.mp4?...",
        # Add more URLs as needed
    ]
  
    # Optional: Custom paths for specific files
    CUSTOM_PATHS = {
        "https://storage.googleapis.com/app.traq365.com/.../video1.mp4?...": "custom/path/video1.mp4"
    }
  
    # Initialize the transfer client with 5 concurrent workers
    transfer_client = TraqToAzureTransfer(
        AZURE_CONNECTION_STRING,
        max_workers=5
    )
  
    # Transfer all files
    results = transfer_client.bulk_transfer(
        url_list=TRAQ_URLS,
        azure_container_name="your-azure-container",
        custom_paths=CUSTOM_PATHS
    )
  
    # Print summary
    print("\nTransfer Summary:")
    print(f"Total Files: {results['summary']['total_files']}")
    print(f"Successful: {results['summary']['successful_transfers']}")
    print(f"Failed: {results['summary']['failed_transfers']}")
    print(f"Success Rate: {results['summary']['success_rate']}")
```

## 5. Usage Instructions

1. **Environment Setup** :

* Install required Python packages: `pip install azure-storage-blob requests tqdm`
* Ensure Python 3.6+ is installed

1. **Configuration** :

* Replace `"your-azure-connection-string"` with the actual Azure connection string
* Set `"your-azure-container"` to your Azure container name
* Populate `TRAQ_URLS` list with the signed URLs from traq.ai

1. **Execution** :

* Run the script: `python transfer_script.py`
* Monitor the progress bar and check the generated log file

1. **Review Results** :

* Examine the console output for a summary
* Check the JSON report file for detailed transfer results

## 6. Important Considerations

1. **URL Expiration** : Ensure traq.ai URLs are valid for the entire transfer duration
2. **Large-Scale Transfers** : May require adjustments for rate limits or extended processing time
3. **Error Handling** : The script logs errors and continues with remaining transfers
4. **Customization** : Adjust `max_workers` for optimal performance based on network conditions

## 7. Next Steps

1. Obtain the comprehensive list of signed URLs from traq.ai
2. Set up Azure Blob Storage and obtain the connection string
3. Configure and test the script with a small subset of videos
4. Review logs and reports to ensure proper functionality
5. Scale up to full transfer after successful testing
6. Implement any necessary error retry mechanisms or monitoring for production use

## 8. Support and Maintenance

For any issues or questions regarding this transfer solution, please contact the development team. Regular updates and maintenance will be performed to ensure compatibility with any changes in the traq.ai or Azure Blob Storage systems.

---

This comprehensive document provides a detailed overview of the video transfer solution, including the full source code, implementation details, and step-by-step instructions for use. It should give your team lead a clear understanding of the solution's capabilities and the process for implementing the video transfer from traq.ai to Azure Blob Storage.
