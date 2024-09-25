---
title: A Complete Guide to Setting up Quartz
draft: false
---

# A Complete Guide to Setting up Quartz
This is a guide on how I configured Quartz and Github Actions to automatically publish notes without ever leaving my note-taking software.

This guide assumes a few things:
- You already have a server you will host Quartz on
- You are using nginx as a web server
- You are using Github to host the source code.
- You are using Obsidian to take notes

This guide can *probably* be adapted to work with web servers other than ngnix and note taking apps other than Obsidian.
# Configuring Obsidian with Git
In order for this set up to work, we need to configure our Obsidian vault to work with Git. If you have already setup your vault to backup to github, you can skip this section. Throughout this guide, I'll be referring to this as the **vault repository** or **notes repository**.

First you will need to install  [Obsidian-Git plugin](https://github.com/Vinzent03/obsidian-git) in Obsidian, which can be installed like any other Obsidian plugin. Don't forget to enable the plugin under *settings > community plugins*

Next, let's set up a brand new github repo. 
- On Github, follow the instructions to [create a new repository from the web UI](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository). Take note of the `.git` link for your repo.
- Next, open your root vault folder (which can be obtained by right clicking a folder in the Obsidian tree and clicking *show in system explorer*) in a terminal. Run the following commands to initialize your repo:
```bash
git init
git add -A
git commit -m "new vault"
git remote add origin git@github.com:<link to your repo ending it .git>
git push -u -f origin main
```


Finally, you will need to configure Obsidian Git with a personal access token so that you can push to your repository.
- To create a personal access token, follow the [github documentation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens). It is very important that you copy this token once it is generated as you will not be able to see the token again.
- In Obsidian, open *settings > Git* and scroll down to the password / personal authentication token field. Paste your personal token in this field.

Once completed, you should be able to run the git commands from the command palette to push, pull, and sync your vault. Optionally, you can configure Obsidian Git to automatically push. If you want to learn more about Obsidian-Git, check out the [documentation](https://publish.obsidian.md/git-doc/Start+here)

# Setting up Quartz
Now that your Obsidian notes are in Github, it is time to set up Quartz. These instructions assume you have a selfhosted server or VPS and are using nginx and that your domain is already configured to point to your machine.

In /var/www/ create a new folder to place quartz and clone the repo to this folder:
```bash
git clone https://github.com/jackyzha0/quartz.git
```

Change into the directory, install packages, and run the setup script.
```bash
cd quartz
npm i
npx quartz create
```
Once done, you can run npx quartz to rebuild your application.

Once done, we can push our Quartz code. Create a new repo through the Github web interface and push your new Quartz folder. This repository will be referred to as the **quartz repository.**

# Configuring the Web Server
In order to make this application visible to the world, you will need to configure nginx. Under `/etc/nginx/sites-available`, create a configuration file with the name of your website if you do not already have one.
```bash
sudo vim /etc/nginx/sites-available/my-site.web
```

Add the following configuration block and save
```
server {

        server_name <your server domain / address>

        root /var/www/<website-name>/quartz/public;
        index index.html;
        error_page 404 /404.html;

        location / {
                try_files $uri $uri.html $uri/ =404;
        }
}
```

Restart nginx with `service nginx restart`.

# Added vault repository as a submodule
In order to have Quartz rebuild each time we create new notes, we need to configure Quartz such that the **content** directory points to a repository of our notes. This can be accomplished using Github submodules.

On our server that is hosting Quartz:
```bash
cd /www/var/<website>/quartz
git submodule add <link to your vault git repo> content
```

Commit and push. Quartz will now build off of the content in your Vault repository.

# Setting up Github Actions to Automate Builds
This is probably the most important part of this entire guide. You might have noticed that when you push to your vault repository, the submodule on Quartz does not automatically update. I found this happening to me and realized it was because the submodule was tied to a particular commit. Not only that, whenever we push notes, Quartz does not automatically build. Lets fix that using Github Actions.

Below is an overview of how the final workflow with Actions will look.

![[Pasted image 20240907185808.png]]

## Step 1: Notify Quartz that our Vault has Changed
First, we need a Github Actions workflow from our **notes repository**. This action needs to do one thing: tell our **quartz repostiory** that changes have occurred. In order to do so, we will be using the **Repository Dispatch** action.

In Github, navigate to your Actions page and select **New Workflow**. Skip the setup and create a workflow yourself. In the workflow file, use the following configuration.
```yaml
name: trigger quartz

on: push

jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
    - name: Repository Dispatch
      uses: peter-evans/repository-dispatch@v3
      with:
        token: ${{secrets.PAT}}
        repository: <your username>/<your quartz repo>
        event-type: rebuild
```

Once the workflow is setup, you will need to add an **actions secret** containing a **repository personal access token**. 
- Create a new personal Access Token, following the instructions provided by [repository dispatch](https://github.com/peter-evans/repository-dispatch)
- Under your repository, go to settings > secrets and variables > actions > new repository secret and create a secret with the name **PAT**.
- Paste the **PAT** that you generated into the secret field and save.

## Step 2: Receive the notification and rebuild Quartz
Now that an event is dispatched to our Quartz repository, we need to make sure that our server does a few things when it receives the notification:
1. ssh into our server
2. Pull an updated version of our **notes** submodule
3. Run `npx quartx build`

To do this, we will be using two actions - `appleboy/ssh-action` and again, `repository-dispatch`. 

The ssh action requires a bit of setup using repository secrets to store ssh keys. In order to set this up, I reccomend following the [instructions provided by ssh-action](https://github.com/appleboy/ssh-action), but in brief:
1. From a computer that **is not your server**, generate a new ssh key using ssh-keygen
2. Add this key to your server's **authorized keys** file
3. Create github secrets for your host name, username, and private host key.

Take note of the names for these secrets, as you will be referencing them in the workflow file. Create a new Github Actions workflow and paste in the below template, replacing the variable names with the ones you chose. Note that the **types** under **repository_dispatch** should match the type you provided in the previous workflow file.

```
name: Repository Dispatch
on: 
  repository_dispatch:
    types: [rebuild]
jobs:
  rebuild:
    runs-on: ubuntu-latest
    steps:
      - name: Create SSH key
        uses: appleboy/ssh-action@v0.1.7
        with: 
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.PRIVATE_HOST_KEY }}
          port: 22
          script_stop: true
          script: cd /var/www/<your site>/quartz && \
		          git submodule update --remote --merge && \
		          npx quartz build
```

Take note that this script will run **everytime** you push changes to your Obsidian vault, regardless of whether or not those notes are marked for publication by **Quartz**. As such, I recommend you limit the number of pushes to when you intend to publish. 

# Wrap Up
With that, you now should have a Quartz instance setup to rebuild each time you push to your Obsidian notes vault. Please let me know if you have any questions, comments, or concerns. If you see an error or run into a problem while following these notes, feel free to reach out.

