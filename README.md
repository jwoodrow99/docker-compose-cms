# Docker Compose CMS

This project is designed to replace shared hosting providers as well as the monolithic CMS architecture. In this project we utilize [Strapi](https://strapi.io/) CMS as a headless CMS, with the intension of creating a React application (Or other framework) and hosting on a platform like Vercel, Netlify or GithubPages.

## Base Configuration

Before we can start the project it requires some very basic configuration. Mainly to allow our domain to be routed to the server.

1. Open ```docker-compose.prod.yml``` and change the "mailserver" container "domainname" setting from ```example.com``` to your actual domain.
2. Open ```docker/config/nginx/conf.d/default.conf``` and ```docker/config/nginx/conf.d/cms.conf.conf``` and replace every ```example.com``` to your actual domain.
3. Add the following DNS records to make everything work:

    | Type | Name |     Content     |
    |:----:|:----:|:---------------:|
    |   A  | @    | \<ServerIP>     |
    |   A  | cms  | \<ServerIP>     |
    |   A  | mail | \<ServerIP>     |
    |  MX  | @    | mail.\<domain>  |
    |  TXT | @    | v=spf1 mx ~all  |

4. Start the docker environment! This will make the base site and the CMS available on port 80 (HTTP, NOT HTTPS). It should be noted that several containers after being created will automatically run install scripts n first run, and will take some time to actually be configured.

    ``` bash
    # Start and Stop docker compose
    docker-compose -f docker-compose.prod.yml up -d --build
    docker-compose -f docker-compose.prod.yml down

    # View status of docker containers
    docker ps -a
    docker logs <container name>

    # Reload Nginx configuration wth zero downtime (Useful for SSL config)
    docker exec docker-compose-cms_nginx_1 nginx -s reload

    # Reset environment
    rm -rf ./docker/volumes
    docker system prune -a
    ```

## SSL Configuration

This project uses certbot to easily install and configure SSL certificates. You can obtain ssl certificates using the "maintenance" container, and will automatically manage certificate renewal. You will want to obtain a certificate for the root domain and each subdomain used. Run the certbot commands replacing ```example.com``` with your actual domain for each of the following.

- example.com
- cms.example.com
- mail.example.com

``` bash
# Test certbot obtaining SSL certificate
docker exec -i docker-compose-cms_maintenance_1 certbot certonly --webroot --webroot-path /var/certbot/ -d example.com --dry-run -v

## Actually obtain certificate
docker exec -i docker-compose-cms_maintenance_1 certbot certonly --webroot --webroot-path /var/certbot/ -d example.com
```

After SSL certificates have been obtained you will need to change the Nginx configuration files to take advantage of HTTPS. Follow the instuctions in the configuration files located at ```docker/config/nginx/conf.d```. Then reload Nginx.

``` bash
docker exec docker-compose-cms_nginx_1 nginx -s reload
```

## MailServer Configuration

For our mail server to actually run, we will need to do further configuration, and add a postmaster account.

1. Create a postmaster account. Replacing "example.com" with your actual domain.
    ``` bash
    docker exec -ti docker-compose-cms_mailserver_1 setup email add postmaster@example.com <password>
    ```
2. Generate DKIM record.
    ``` bash
    docker exec -ti docker-compose-cms_mailserver_1 setup config dkim
    ```
3. View the generated DKIM record and add it to the DNS records. Replacing "example.com" with your actual domain.
    ``` bash
    cat ./docker/volumes/mailserver/config/opendkim/keys/example.com/mail.txt
    ```

    | Type |       Name      |                       Content                                                                             |
    |:----:|:---------------:|:---------------------------------------------------------------------------------------------------------:|
    |  TXT | mail._domainkey | v=\<value>;  h=\<value>;  k=\<value>;  p=\<value>;                                                        |
    |  TXT | _dmarc          | v=DMARC1; p=none; rua=mailto:postmaster@example.com; ruf=mailto:postmaster@example.com; sp=none; ri=86400 |

4. Connect your mail client using the following configuration
   
    Incomming IMAP:
    |    Key   |      Value        |
    |:--------:|:-----------------:|
    | Hostname | mail.example.com  |
    | Port     | 143               |
    | Security | STARTTLS          |

    Outgoing SMTP:
    |    Key   |      Value        |
    |:--------:|:-----------------:|
    | Hostname | mail.example.com  |
    | Port     | 587               |
    | Security | STARTTLS          |
