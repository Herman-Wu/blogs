


- Action Deployed Failed: 
    - GitHub key failed  
    - [Create SSH Deploy Key](https://github.com/peaceiris/actions-gh-pages#%EF%B8%8F-create-ssh-deploy-key) in a Linux environment

    ```bash 
    ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
    ```

        Your identification has been saved in gh-pages
        Your public key has been saved in gh-pages.pub
        The key fingerprint is:
        SHA256:Pewy7cgzP317vBhdalU0wyv6wQXOiig85luYSmwm760 herman.wu@live.com

    - Go to the project's Repository Settings
        - Go to Deploy Keys and add your public key with the Allow write access
        - Go to Secrets and add your private key as ACTIONS_DEPLOY_KEY

