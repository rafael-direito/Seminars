
# GitHub Actions Demonstration

For this demonstration we will deploy a [docusaurus](https://docusaurus.io/) instance using GitHub Actions.

# 1. Create the GitHub Repository

First of all, you'll have to create a GitHub Repository to host your code. Then, clone this repository to your local VM.

# 2. Install Docusaurus

## 2.1 What is docusauros?

Docusaurus is a **static-site generator**. It builds a **single-page application** with fast client-side navigation, leveraging the full power of **React** to make your site interactive. It provides out-of-the-box **documentation features** but can be used to create **any kind of site** (personal website, product, blog, marketing landing pages, etc).

## 2.2 Installation

Before creating a new docusaurus website, you'll have to install Node.js version 14.13 or above. To do so, follow its official documentation.

After installing Node.js, you can validate its version through `node -v`

Now, you can navigate to the local folder where you have cloned your GitHub Repository. In my case this folder is named `github-actions-demo`.

Then, bootstrap a docusaurus website: `npx create-docusaurus@latest project-documentation classic`. Here, the `project-documentation` is the name of the website.

After executing that command, you should be faced with a new directory `project-documentation`.

It's structure should be something like this one:

``` 
project-documentation  
├── blog  
│ ├── 2019-05-28-hola.md  
│ ├── 2019-05-29-hello-world.md  
│ └── 2020-05-30-welcome.md  
├── docs  
│ ├── doc1.md  
│ ├── doc2.md  
│ ├── doc3.md  
│ └── mdx.md  
├── src  
│ ├── css  
│ │ └── custom.css  
│ └── pages  
│ ├── styles.module.css  
│ └── index.js  
├── static  
│ └── img  
├── docusaurus.config.js  
├── package.json  
├── README.md  
├── sidebars.js  
└── yarn.lock
```

## 2.3 Run Docusaurus

To run our website, execute the following command from within the `project-documentation` folder : `npm run start`

You should be presented with a message similar to this one: 
`[SUCCESS] Docusaurus website is running at http://localhost:3000/.`


# 3. Install a GitHub Self-Hosted Runner, which will (test and) build your website.

Since we only want our website to run inside our infra-strucure (our documentation will be considered confidential), we will deploy a GitHub Self-Hosted Runner, in order to build and deploy the website inside a VM that is not exposed to the world.

To do so, follow this tutorial: [Adding self-hosted runners](https://docs.github.com/pt/actions/hosting-your-own-runners/adding-self-hosted-runners) 

Stop before executing the run.sh script. We will execute this script inside a `screen`so we can check its logs all the time.

Let's do it then:

``` bash
screen -S github-runner # create the screen
# then, we will be presented with a new session
# inside this section run
./run.sh
# After running this command, you should see the following message "Connected to GitHub"
# No exit the screen
# To do so, press ctrl + a + d
# If you want to re-attach the screen, execute 
# screen -r github-runner
# When doing this you should see the session that was created by the first command.
``` 

# 4. Create a GitHub Workflow

At the root of your repository execute the following commands:
``` bash
mkdir -p .github/workflows
cd .github/workflows
touch deployment-workflow.yml
```

Now, add the following content to your `deployment-workflow.yml` file:

``` yaml
```yaml
name: GitHub Actions Demo
on:
  push:
    branches: [ main ]
jobs:
  Explore-GitHub-Actions:
    runs-on: self-hosted
    steps:
      - run: echo "🎉 The job was automatically triggered by a ${{ github.event_name }} event."
      - run: echo "🐧 This job is now running on a ${{ runner.os }} server hosted by GitHub!"
      - run: echo "🔎 The name of your branch is ${{ github.ref }} and your repository is ${{ github.repository }}."
      - name: Check out repository code
        uses: actions/checkout@v3
      - run: echo "💡 The ${{ github.repository }} repository has been cloned to the runner."
      - run: echo "🖥️ The workflow is now ready to test your code on the runner."
      - name: List files in the repository
        run: |
          ls ${{ github.workspace }}
      - run: echo "🍏 This job's status is ${{ job.status }}."
```

Commit this file to your repository and check if the workflow is being executed.

In your GitHub Repository you should see that the action is being processed.

![action being processed](https://i.imgur.com/mG7p6UQ.png)

You can also see what is being executed in the workflow.

![Workflow Output](https://i.imgur.com/a98uOQb.png)

Also, if you check the screen where your runner is running you should see the following:

![screen](https://i.imgur.com/w5FfCNP.png)


---
Now, we have to define what we want to do with our workflow!
In this demo, we have a NGINX Server deployed on the same VM as the GitHub Self-Hosted Runner. We will use this web serve our static site. So, the NGINX will retrieve the static file from a specific location and serve them to the end-users.

To get this static files (static website), you'll have to build your docusaurus project.

Inside your docusaurus project, run the following command: `npm run build`. After a few seconds the build process will end, and you'll be presented with a `build` folder. There you will find the rendered static files which NGINX will serve.

Now, let's check if everything is ok with our build. To do so, we will run a python simple HTTP server.

```
cd build
python3 -m http.server 8080
```

This will start a development web server in port 8080. Open your browser and check if the files are ok.

We can see the website is running!

![My Website](https://i.imgur.com/9NAtSKm.png)

Now we have to recall that NGINX will have to serve these pages, so we will have to provide this files in a location where NGINX can find them. We will have to create a new directory in our VM - `/var/www/docusaurus-website`  ( `mkdir -p /var/www/docusaurus-website`).

Then, we will have to configure our Self Hosted Agent to:

1 - Build the docusaurus website
2 - Just in case, make a backup of the older version of the website that was deployed.
3 - Move the files to the folder we have just created

We will store the backup files in the following directory `/var/www/docusaurus-website-backups` (you'll need to create it).


For now, let's start with something simple. Update your `deployment-workflow.yml` file, so it matches this file:

```yaml
name: GitHub Actions Deployment Workflow
on:
  push:
    branches: [ main ]
jobs:
  build_website:
    runs-on: self-hosted
    steps:
      - run: | # get inside the project and deal with the dependencies
          cd ${{ github.workspace }}/project-documentation 
          if [ -e yarn.lock ]; then
            yarn install --frozen-lockfile
          elif [ -e package-lock.json ]; then
            npm ci
          else
            npm i
          fi
      - run: |
          cd ${{ github.workspace }}/project-documentation 
          npm run build
```

Be sure if have Node.Js installed on your Runner's VM (same version) !

Now, check the GitHub Actions Tab to verify if the build was successful:

![Second Build](https://i.imgur.com/QtbHvHD.png)

Great, the project was rendered successful!

We can now update our GitHub Workflow to the following:

```yaml
name: GitHub Actions Deployment Workflow
on:
  push:
    branches: [ main ]
jobs:
  build_website:
    runs-on: self-hosted
    steps:
      - run: | # get inside the project and deal with the dependencies
          cd ${{ github.workspace }}/project-documentation 
          if [ -e yarn.lock ]; then
            yarn install --frozen-lockfile
          elif [ -e package-lock.json ]; then
            npm ci
          else
            npm i
          fi
      - run: |
          cd ${{ github.workspace }}/project-documentation 
          npm run build

  crete_backup:
    needs: build_website
    runs-on: self-hosted
    steps:
      - run: | # create a zip file with all the files and store it in the backups folder
          tar -czvf /var/www/docusaurus-website-backups/backup-$(date '+%Y-%m-%d_%H:%M:%S').tar.gz /var/www/docusaurus-website
      - run: | # remove all backups older than 7 days
          find /var/www/docusaurus-website-backups/ -maxdepth 1 -type f -mtime +7 -print | xargs /bin/rm -f

  deploy_website:
    needs: crete_backup
    runs-on: self-hosted
    steps:
      - run: | # delete old website files
          rm -rf /var/www/docusaurus-website/*
      - run: | # add the new ones
          cp -r ${{ github.workspace }}/project-documentation/build/* /var/www/docusaurus-website/
```

Notice the `needs: <job>` tag. These tags are used to define a sequence for executing each job. If you don't do this, several jobs can be executed in parallel.

After committing this new workflow, if we go to [atnog-cicd-classes2.av.it.pt/](https://atnog-cicd-classes2.av.it.pt/) we should be able to see our docusaurus website.

Now, lets add a few more READMEs!

Inside your docusaurus project execute the following command: `mkdir -p docs/my-tutorials`
Then, inside this folder, create the following files:

--- 

`_category_.json`
``` javascript 
{
  "label": "My New Tutorials",
  "position": 4,
  "link": {
    "type": "generated-index"
  }
}
```

---

`README1.md`
``` markdown 
---
sidebar_position: 1
---

# README 1

This is my first README!!!
```

---

`README2.md`
``` markdown 
---
sidebar_position: 2
---

# README 2

This is my first README!!!
```

---

Now, commit these new files to your GitHub repository. Wait for the workflow to run and open the website. There you should see these new files.

![My New Tutorials](https://i.imgur.com/ePyPidv.png)


# Conclusions

This demonstration presented a brief introduction to GitHub Actions using Self Hosted Runners. You have a plethora of tutorials, available online, that can help you with the creation of a complete CI/CD Pipeline. You can use GitHub Runners during your project, or a similar tool. It is required you have a pipeline to build, test and deploy your application.

For more information contact  [rdireito@av.it.pt](mailto:rdireito@av.it.pt)
