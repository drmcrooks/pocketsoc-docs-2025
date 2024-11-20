# Access

- We have enough instances for you to work with.

- Each user is able to access their individual containers.

- The connection goes through the available proxy host, which then redirects you to the relevant application.

- Here's how you can access your applications:
    - Suppose you have been assigned `node-1`.
    - To access **Portainer**, append the following to the URL:

          ```  
          host-192-168-1-100.something.com:1001 
          ```

    - Similarily, to access **OpenSearch**, append the following to the URL:
    
          ```  
          host-192-168-1-100.something.com:1002 
          ```

    - Finally, to access **MISP**, append the following to the URL: 

          ``` 
          host-192-168-1-100.something.com:1003 
          ```

- If you have been assigned `node-2`, you will be using the ports `2001`, `2002`, `2003`
