# Generate some traffic

The traffic you can see already is background - DNS requests, etc. We now want to generate some traffic of our own that we can identify

1. In a Portainer tab, look for the client container and open a terminal
2. Do a simple `curl` command to download a "payload" from our webserver  
    ```
    curl http://webserver -o payload  
    ```
3. Let's check the MD5 checksum of this file   
   ```
    md5sum payload  
   ```  
4. This forms an extremely simple analysis, but gives us enough to go back to the Dashboards tab  
5. Before continuing, take a note of the IP address of the webserver, `dig webserver`  
6. If you would like to pull *benign* data from an external source into another payload file, also take a note of the checksum that is generated  