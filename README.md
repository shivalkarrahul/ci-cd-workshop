# **ci-cd-workshop**

## **1. Access the Workshop Repository**

You can open the CI/CD Workshop repository in any of the following ways:

### **Method 1: Google Search**

1. Visit **[https://www.google.com](https://www.google.com)**
2. Search for **rahul shivalkar github**
3. Open the GitHub profile shown in the results, go to Repositories.
4. Click the repository **ci-cd-workshop**.

![GitHub Repo List](artifacts/1-github-repo-list.png)


### **Method 2: Direct Links**

* GitHub Profile: **[https://github.com/shivalkarrahul](https://github.com/shivalkarrahul)**
* Workshop Repo: **[https://github.com/shivalkarrahul/ci-cd-workshop](https://github.com/shivalkarrahul/ci-cd-workshop)**

---

## **2. Create or Sign In to Your AWS Account**

1. Open: **[https://aws.amazon.com/console/](https://aws.amazon.com/console/)**
2. If you don’t have an account → **Create a new AWS account**
3. If you already have one → **Sign In**

![AWS Console](artifacts/2-aws-console.png)
---

## **3. Create or Sign In to Your GitHub Account**

1. Open: **[https://github.com/](https://github.com/)**
2. If you don’t have an account → **Sign up**
3. If you already have an account → **Sign in**

![GitHub Console](artifacts/3-github-console.png)

---

## **4. Launch the Build Server (EC2 Instance)**

We will create **one build server** in the **N. Virginia (us-east-1)** region.

### **Step 4.1 — Confirm AWS Region**

Ensure your AWS region is set to:

**US East (N. Virginia) — us-east-1**

### **Step 4.2 — Create the EC2 Instance**

1. In the AWS Console, search for **EC2** and open it.

2. Click **Instances** in the left-hand menu.

3. Click **Launch instance**.

4. Fill in the following configuration details:

   * **Name:** `ci-cd-workshop-build-server`

   * **AMI:** Ubuntu Server 24.04 LTS (HVM), SSD Volume Type

   * **Instance Type:** `t2.medium`

   * **Key Pair:**

     * Click **Create new key pair**
     * **Name:** `ci-cd-workshop`
     * **Type:** RSA
     * **File Format:** `.pem`
       *(We will use AWS CloudShell, which supports `.pem` files directly.)*
     * Click **Create key pair** to download the file.

   * **Network Settings:**

     * Select **Create security group**
     * Allow SSH (port 22) from **Anywhere (0.0.0.0/0)**
     * ⚠️ **This is NOT recommended for production.**
       For demo and workshop purposes, we are allowing open SSH access to avoid connection issues.

   * **Storage:**

     * Root volume size: **20 GB**

   * **Number of Instances:**

     * Set to **1**

5. Click **Launch instance**.

![Build Server](artifacts/4-build-server.png)


## **4.1 Create IAM Role for Build Server EC2**

To allow the backend to access S3 :


1. Go to **AWS Console → IAM**: [https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/home](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/home)
2. Role
3. Click **Create role**
4. Choose **EC2** as the Service or use case →  Next

5. Search and Attach the following policies →  Next

   * `AmazonS3FullAccess`
6. Name the role: `ci-cd-workshop-build-server-role`
7. Create role

## **4.2 Attach the IAM Role to Build Server EC2 Instance**
1. Go to **EC2 → Instances → Select ci-cd-workshop-build-server → Actions → Security → Modify IAM Role → Search and Select `ci-cd-workshop-build-server-role` → Click Update IAM role**


## **5. Connect to the Build Server (EC2 Instance)**

When launching the EC2 instance earlier, we downloaded a **.pem** key pair file.
This key is required to securely connect to the server.

There are multiple ways to connect to an EC2 instance:

* **Windows:** Use PuTTY
  → You would download a **.ppk** key (or convert `.pem` → `.ppk`)
* **Mac / Linux:** Use the Terminal (supports `.pem` directly)
* **AWS CloudShell:** Easiest method, works from the browser

For this workshop, **we will use AWS CloudShell** to keep things simple.

---

### **Step 5.1 — Open AWS CloudShell**

1. On the AWS Console header, click the **CloudShell icon**
   (located near the **top center** of the page, slightly right of the search box).

![CloudShell Icon](artifacts/5-cloudshell-icon.png)

2. A terminal window will open at the bottom of your browser.
3. CloudShell will appear regardless of which AWS page you are on.

![CloudShell Screen](artifacts/6-cloudshell-screen.png)


---

### **Step 5.2 — Upload Your .pem File**

1. In CloudShell, click **Actions → Upload file**
2. Select the key file you downloaded earlier:
   **`ci-cd-workshop.pem`**
3. Upload the file.

![Upload PEM Key](artifacts/7-upload-pem-key.png)

---

### **Step 5.3 — Set Correct File Permissions**

Run the following command in CloudShell:

```bash
chmod 400 ci-cd-workshop.pem
```

This sets secure permissions required by SSH.

---

### **Step 5.4 — Connect to the ci-cd-workshop-build-server EC2 Instance**

1. Open the **EC2 Console**
2. Select your instance **ci-cd-workshop-build-server**
3. Copy the **Public IPv4 address** from the **Details** tab

Now connect to your server:

```bash
ssh -i ci-cd-workshop.pem ubuntu@<PUBLIC-IP>
```

When asked:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type:

```
yes
```

---

### You are now inside the Build Server.

![SSH into Build Server](artifacts/8-ssh-in-build-server.png)

### **Step 5.5 — Install AWS CLI on Build Server**

Run the following commands on the build server:

```bash
sudo apt update -y
```

```bash
sudo apt install unzip curl -y
```

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```bash
unzip awscliv2.zip
```

```bash
sudo ./aws/install
```

```bash
aws --version
```

✅ After this, the `aws` command is available and pipelines can use it to deploy the frontend to S3.

---

## **6. Install Jenkins on the Build Server (EC2 Instance)**

Follow the steps below on your EC2 build server after connecting through CloudShell.

---

### **1. Install Java (Required by Jenkins)**

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

Verify:

```bash
java -version
```

---

### **2. Add the Jenkins Repository Key**

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key \
  | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

---

### **3. Add the Jenkins Repository**

```bash
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

---

### **4. Update apt**

```bash
sudo apt update
```

You should now see entries from **pkg.jenkins.io** in the output.

---

### **5. Install Jenkins**

```bash
sudo apt install jenkins -y
```

---

### **6. Start and Enable Jenkins Service**

Start Jenkins:

```bash
sudo systemctl start jenkins
```

Enable auto-start on boot:

```bash
sudo systemctl enable jenkins
```

Check status:

```bash
sudo systemctl status jenkins
```

You should see **active (running)**.

---

### **7. Allow Jenkins Port in Security Group**

Jenkins runs on **port 8080**, so we must allow inbound traffic to this port on the Build Server’s security group.

Follow these steps carefully:

1. Go to the **AWS Console**

2. Search for **EC2** and open it

3. In the left menu, click **Instances**

4. Select your instance named **ci-cd-workshop-build-server**

5. At the bottom panel, click on the **Security** tab

6. Under **Security groups**, click on the security group linked to your instance
   (Example: `sg-0123456789abcdef`)

7. Now you are on the **Security Group Details** page

8. Click the **Inbound rules** tab

9. Click on **Edit inbound rules**

10. Click **Add rule**

11. Enter the following:

    * **Type:** Custom TCP
    * **Port range:** `8080`
    * **Source:** `0.0.0.0/0` (Anywhere IPv4)

      > This is **NOT recommended for production**, but acceptable for this workshop/demo.

12. Click **Save rules**

Your Build Server is now accessible on port **8080**, which is required to open the Jenkins UI.

![SG 22 & 8080 Rule](artifacts/9-sg-22-8080-rule.png)

---

### **8. Access Jenkins UI**

Open your browser and visit:

```
http://<public-ip>:8080
```

Replace `<public-ip>` with the public IP of the Build Server EC2 instance.

---

### **9. Retrieve Initial Admin Password**

Run the following command in CloudShell (already connected to your EC2 instance):

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the displayed password and paste it into the Jenkins unlock screen.

Here is **Step 10** written clearly and professionally in the same workshop style:

---

### **10. Complete Jenkins Initial Setup**

After unlocking Jenkins, you will see the setup wizard. Follow these steps:

1. **Install Suggested Plugins**
   Jenkins will automatically begin installing the recommended plugins.
   (This may take a few minutes.)

2. On the next screen, when asked to create the first admin user:
   Click **“Skip and continue as admin”**
   (For the workshop, we do not need to create a new user.)

3. Jenkins will show the **Jenkins URL**.
   Do **not** change anything here.
   Simply click **Save and Finish**.

4. You will now see the confirmation message:
   **“Jenkins is ready!”**

5. Click **Start using Jenkins**

You will now be taken to the Jenkins dashboard.

![Jenkins UI](artifacts/10-jenkins-ui.png)

### **10.1. Install Required Plugin — SSH Agent Plugin**

The backend pipeline uses the sshagent step for SSH-based deployment.
To enable it, you must install the SSH Agent Plugin.

111
Here is a clean, clear, updated version **including the SSH Agent plugin installation step**:

---

# **10. Complete Jenkins Initial Setup**

After unlocking Jenkins, you will see the setup wizard. Follow these steps:

---

## **1. Install Suggested Plugins**

Jenkins will automatically begin installing the recommended plugins.
(This may take a few minutes.)

---

## **2. Skip Admin User Creation**

When Jenkins asks you to create the first admin user:

* Click **“Skip and continue as admin”**

(For this workshop, you do not need to create a separate user.)

---

## **3. Confirm Jenkins URL**

Jenkins will show the default URL.

* Do **not** change anything
* Click **Save and Finish**

---

## **4. Jenkins Ready Screen**

You will now see:

**“Jenkins is ready!”**

* Click **Start using Jenkins**

This will take you to the Jenkins dashboard.

# **⚡ 10.1 Install Required Plugin — SSH Agent Plugin**

The backend pipeline uses the `sshagent` step for SSH-based deployment.
To enable it, you must install the **SSH Agent Plugin**.

1. From the Jenkins dashboard → Click **Manage Jenkins**
2. Click **Plugins**
3. Go to the **Available Plugins** tab
4. Search for:

```
SSH Agent
```

5. Install **SSH Agent Plugin**
6. Restart Jenkins when prompted

This plugin enables the `sshagent { ... }` syntax used in the backend Jenkinsfile.

![Jenkins UI](artifacts/10.1-jenkins-install-ssh-agent.png)

---

## **7. Create Infrastructure for 3-Tier App**

Before we deploy our code, we need to create the **infrastructure** where the app will run. This includes:

1. **Frontend** → S3 bucket (static website)
2. **Backend** → EC2 instance (Flask app)
3. **Database** → DynamoDB table

Let’s create them step by step.

---

Got it! Here’s a **single-step version** that includes bucket creation, static website hosting, and the public access policy:

---

### **1️⃣ Create Frontend S3 Bucket (Static Website)**

1. Open the **AWS Console → S3**: [https://us-east-1.console.aws.amazon.com/s3/home?region=us-east-1](https://us-east-1.console.aws.amazon.com/s3/home?region=us-east-1) and click **Create bucket**.
2. Configure the bucket:

   * **Bucket name:** `ci-cd-workshop-frontend-<your-name>`
   * **Region:** same as your EC2 Build Server (e.g., **us-east-1**)
   * **Block all public access:** **Uncheck** and tick the acknowledgement checkbox

     > ⚠ Note: Not recommended for production, but fine for this demo. Bucket names must be globally unique — if taken, add some characters at the end.
   * Leave other options as default.
3. Click **Create bucket**.
4. Enable **Static website hosting**:

   * Open your bucket → **Properties → Static website hosting → Edit → Enable**
   * Index document: `index.html` → **Save changes**
   * Note the **Bucket website endpoint URL**.
5. Make the bucket publicly accessible:

   * Go to **Permissions → Bucket Policy → Edit**
   * Paste the following policy (replace `<your-bucket-name>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<your-bucket-name>/*"
    }
  ]
}
```

6. Click **Save changes**.

✅ Your frontend S3 bucket is ready to host the static website and is publicly accessible.

---

### **2️⃣ Create Backend EC2 Instance**

Our backend Flask app will run on an EC2 instance.

1. Go to **AWS Console → EC2**: [https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Home:](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Home:)
2. Click **Launch instances**
3. Configure the instance:

   * **Name:** `ci-cd-workshop-backend-server`
   * **AMI:** Ubuntu Server 24.04 LTS (HVM), SSD Volume Type
   * **Instance type:** t2.medium
   * **Key pair:** Select an existing `ci-cd-workshop`
   * **Network / Security group:**

     * Allow **SSH (22)** from **Anywhere (0.0.0.0/0)**

       > ⚠ Not recommended for production, but fine for our workshop
   * **Storage:** 20 GB
   * **Number of instances:** 1
4. Click **Launch instance**
5. Once running, note the **public IP** — we will use this to connect to the backend.

### **3. Allow Python Port in Security Group**

Python runs on **port 5000**, so we must allow inbound traffic to this port on the Backend Server’s security group.

Follow these steps carefully:

1. Go to the **AWS Console**

2. Search for **EC2** and open it

3. In the left menu, click **Instances**

4. Select your instance named **ci-cd-workshop-backend-server**

5. At the bottom panel, click on the **Security** tab

6. Under **Security groups**, click on the security group linked to your instance
   (Example: `sg-0123456789abcdef`)

7. Now you are on the **Security Group Details** page

8. Click the **Inbound rules** tab

9. Click on **Edit inbound rules**

10. Click **Add rule**

11. Enter the following:

    * **Type:** Custom TCP
    * **Port range:** `5000`
    * **Source:** `0.0.0.0/0` (Anywhere IPv4)

      > This is **NOT recommended for production**, but acceptable for this workshop/demo.

12. Click **Save rules**

Your Backend Server is now accessible on port **5000**, which is required for the Frontend App to connect on the Backed App.

![SG 22 & 5000 Rule](artifacts/10.2-sg-22-5000-rule.png)


### **4 Connect to the ci-cd-workshop-backend-server EC2 Instance**

📌 Important:
Before connecting, ensure you're working from AWS CloudShell, not from the ci-cd-workshop-build-server EC2 instance.
If unsure, simply close CloudShell and reopen it—this will start a fresh CloudShell session.

1. Open the **EC2 Console**
2. Select your instance **ci-cd-workshop-backend-server**
3. Copy the **Public IPv4 address** from the **Details** tab

Now connect to your server:

```bash
ssh -i ci-cd-workshop.pem ubuntu@<PUBLIC-IP>
```

When asked:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type:

```
yes
```

---

### You are now inside the Build Server.

![SSH into Build Server](artifacts/8-ssh-in-build-server.png)

### **Step 5.5 — Install AWS CLI on Build Server**

Run the following commands on the build server:

```bash
sudo apt update -y
```

```bash
sudo apt install unzip curl -y
```

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```bash
unzip awscliv2.zip
```

```bash
sudo ./aws/install
```

```bash
aws --version
```
---

### **3️5 Create Database Layer (DynamoDB)**

We will create **two DynamoDB tables** to store all app data for the workshop:

1. **Assignments Table** – stores assignment metadata

   * **Table name:** `ci-cd-workshop-assignments`
   * **Primary key:** `assignment_id` (String)

2. **Submissions Table** – stores student submissions

   * **Table name:** `ci-cd-workshop-submissions`
   * **Primary key:** `submission_id` (String)

> ⚠ Note: DynamoDB table names must be unique in your account. If you already have a table with this name, add a short suffix like your initials (e.g., `ci-cd-workshop-assignments-rs`)

---

### **6Steps to Create the Tables**

1. Open **AWS Console → DynamoDB**: [https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#dashboard](https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#dashboard)
2. Click **Create table** for the **Assignments Table**:

   * Enter **Table name:** `ci-cd-workshop-assignments`
   * Set **Partition key:** `assignment_id` (String)
   * Leave all other settings as default (on-demand capacity, encryption, etc.)
   * Click **Create table**
3. Repeat the process for the **Submissions Table**:

   * Enter **Table name:** `ci-cd-workshop-submissions`
   * Set **Partition key:** `submission_id` (String)
   * Leave all other settings as default → Click **Create table**

✅ Both tables are now ready to store **assignments** and **student submissions** for the workshop app.

---

### **7 Create IAM Role for ci-cd-workshop-backend-server EC2**

To allow the backend to access S3 and DynamoDB:


1. Go to **AWS Console → IAM**: [https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/home](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/home)
2. Role
3. Click **Create role**
4. Choose **EC2** as the Service or use case →  Next

5. Search and Attach the following policies →  Next

   * `AmazonS3FullAccess`
   * `AmazonDynamoDBFullAccess`
6. Name the role: `ci-cd-workshop-backend-server-role`
7. Create role

### **5 Attach the IAM Role to EC2**
1. Go to **EC2 → Instances → Select ci-cd-workshop-backend-server → Actions → Security → Modify IAM Role → Search and Select `ci-cd-workshop-backend-server-role` → Click Update IAM role**

---

✅ All three tiers of our app are now ready!

---

## **8. Set Up GitHub Repositories and Push Workshop Files**

In this step, we will **create GitHub repositories**, clone them on the **build server**, copy the prepared workshop files, commit, and push them to GitHub using a **Personal Access Token (PAT)**.

> ⚠️ All commands should be run **on the build server** using CloudShell or SSH.

---

### **1 — Create GitHub Repositories**

1. Open your GitHub account: [https://github.com/](https://github.com/)
2. Click **New**.
3. Create **two repositories**:

   * `ci-cd-workshop-frontend`
   * `ci-cd-workshop-backend`
4. Leave all options default (Public, No README, No .gitignore, No license).
5. Click **Create repository**.

![Create Github Repos](artifacts/11-create-new-repo.png) 

---

### **2 — Create GitHub Personal Access Token (PAT)**

1. Click your **profile picture → Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token → Generate new token (classic)**
3. Give a **note/name**: `ci-cd-workshop-token`
4. Select **expiration** (e.g., 90 days)
5. Under **Scopes**, select:

   * `repo` → Full control of private and public repositories
   * `workflow` → Optional for Jenkins triggers (future step)
6. Click **Generate token**
7. Copy the token somewhere safe, do not share with anyone. **You won’t be able to see it again**.

> We will use this token as the password when pushing commits from the build server.

![Create Github Token](artifacts/12-create-token.png)

---

### **3 — Connect to Build Server**

1. Open **CloudShell** from AWS Console:

```bash
ssh -i ci-cd-workshop.pem ubuntu@<BUILD-SERVER-PUBLIC-IP>
```

2. Navigate to a working directory:

```bash
mkdir ~/ci-cd-workshop
```

```bash
cd ~/ci-cd-workshop
```

---

### **4 — Configure Git (First Time Only)**

```bash
git config --global user.name "Your Name"
```

```bash
git config --global user.email "your-email@example.com"
```

---

### **5 — Clone the Repositories Using HTTPS**

Replace `<your-username>` with your GitHub username:

```bash
# Frontend
git clone https://github.com/<your-username>/ci-cd-workshop-frontend.git
```

```bash
# Backend
git clone https://github.com/<your-username>/ci-cd-workshop-backend.git
```

> When prompted for username and password, **use your GitHub username** and **Personal Access Token** as password.

![Clone Github Repos](artifacts/13-clone-repo.png)

![Clone Output](artifacts/14-clone-output.png)


---

### **6 — Download and Copy Workshop Files**

#### **Frontend Files**

```bash
cd ~/ci-cd-workshop/ci-cd-workshop-frontend

wget https://raw.githubusercontent.com/shivalkarrahul/ci-cd-workshop/main/frontend/index.html
wget https://raw.githubusercontent.com/shivalkarrahul/ci-cd-workshop/main/frontend/frontend_version.json
wget https://raw.githubusercontent.com/shivalkarrahul/ci-cd-workshop/main/frontend/Jenkinsfile


```

#### **Backend Files**

```bash
cd ~/ci-cd-workshop/ci-cd-workshop-backend

wget https://raw.githubusercontent.com/shivalkarrahul/ci-cd-workshop/main/backend/app.py
wget https://raw.githubusercontent.com/shivalkarrahul/ci-cd-workshop/main/backend/backend_version.txt
wget https://raw.githubusercontent.com/shivalkarrahul/ci-cd-workshop/main/backend/requirements.txt
wget https://raw.githubusercontent.com/shivalkarrahul/ci-cd-workshop/main/backend/Jenkinsfile
```

---

### **7 — Commit and Push Changes**

#### **Frontend Repo**

```bash
cd ~/ci-cd-workshop/ci-cd-workshop-frontend
```

```bash
ls -l
```

```bash
git status
```

```bash
git add .
```

```bash
git status
```

```bash
git commit -m "Initial commit: workshop frontend files"
```

```bash
git push origin main
```

![Push to Fronend Repo](artifacts/15.1-push-to-frontend-repo.png)

#### **Backend Repo**

```bash
cd ~/ci-cd-workshop/ci-cd-workshop-backend
```

```bash
ls -l
```

```bash
git status
```

```bash
git add .
```

```bash
git status
```

```bash
git commit -m "Initial commit: workshop backend files"
```

```bash
git push origin main
```

![Push to Backend Repo](artifacts/15.1-push-to-backend-repo.png)



> During `git push`, Git will prompt for credentials:
>
> * **Username:** GitHub username
> * **Password:** Personal Access Token (PAT) created earlier

---

### **8 — Verify on GitHub**

1. Open the repositories in your GitHub account:

   * `https://github.com/<your-username>/ci-cd-workshop-frontend`
![Changes in Fronend Repo](artifacts/15.2-changes-in-frontend-repo.png)

   * `https://github.com/<your-username>/ci-cd-workshop-backend`
![Changes in Backend Repo](artifacts/15.2-changes-in-backend-repo.png)

2. You should see all your files and initial commits.

---

✅ **Congratulations!** Your workshop files are now versioned in GitHub and ready for Jenkins CI/CD pipelines.

---

## **9. Configure Jenkins Pipelines & Deployment (CI/CD Automation)**

In this step, we will **configure Jenkins** to automatically deploy the **frontend** to S3 and the **backend** to the backend EC2 server whenever code is pushed to GitHub. We will also store the existing EC2 key pair in Jenkins for secure SSH access.

> ⚠️ All commands and steps are performed on the **build server** via CloudShell or SSH.

---

### **1 — Store SSH Key in Jenkins**

We will use the **existing EC2 key pair** `ci-cd-workshop.pem` for backend deployment.

1. Open your **Jenkins Dashboard** in browser:

   ```
   http://<build-server-public-ip>:8080
   ```
2. Go to **Manage Jenkins(Gear Icon in right top corner) → Credentials → System → Global credentials → Add Credentials**.
3. Open `ci-cd-workshop.pem` in notepad/textpad on your local machine and copy the content
4. Select:

   * **Kind:** SSH Username with private key
   * **Scope:** Global
   * **ID:** `backend-server-ssh`
   * **Description:** SSH Key for backend-server deployment
   * **Username:** `ubuntu`
   * **Private Key:** Enter directly → paste the content of `ci-cd-workshop.pem` you copied
   
5. Click **Create**.

✅ Jenkins now has the SSH key to connect to the backend server.

![SSH Jenkins Credentials](artifacts/16-ssh-jenkins-credentials.png)

---

### **2 — Create Jenkins Pipeline for Frontend**

We will create a **frontend pipeline** that deploys the static site to S3 using a **parameterized bucket name**.

1. In Jenkins Dashboard → **New Item**.
2. Enter **Item Name:** `frontend-deploy`
3. Select **Pipeline** → Click **OK**.
4. General → Tick **GitHub project**, Checkbox and add repo URL in Project url.
5. Triggers → Tick **GitHub hook trigger for GITScm polling** Checkbox
6. Scroll to **Pipeline section → Definition:** `Pipeline script from SCM`.
7. Select **Git** → Repository URL:

   ```
   https://github.com/<your-username>/ci-cd-workshop-frontend.git
   ```
8. Branch: `main`
9. Script Path: `Jenkinsfile`
10. Save the pipeline.

> **Note:** The frontend `Jenkinsfile` should use a parameter for bucket name, so when triggered via webhook, it automatically deploys to the correct S3 bucket without prompting.

![Frontend Backend Pipeline](artifacts/17-frontend-backend-pipelines.png)

---

### **3 — Create Jenkins Pipeline for Backend**

We will create a **backend pipeline** that deploys the Python Flask app to the backend EC2 server using SSH.

1. In Jenkins Dashboard → **New Item**.
2. Enter **Item Name:** `backend-deploy`
3. Select **Pipeline** → Click **OK**.
4. General → Tick **GitHub project**, Checkbox and add repo URL in Project url.
5. Triggers → Tick **GitHub hook trigger for GITScm polling** Checkbox
6. Scroll to **Pipeline section → Definition:** `Pipeline script from SCM`.
7. Select **Git** → Repository URL:

   ```
   https://github.com/<your-username>/ci-cd-workshop-backend.git
   ```
8. Branch: `main`
9. Script Path: `Jenkinsfile`
10. Save the pipeline.

> **Note:** Backend `Jenkinsfile` uses the stored SSH key (`backend-server-ssh`) to connect to backend EC2, setup virtual environment, install requirements, stop old process, and start Flask app.

![Frontend Backend Pipeline](artifacts/17-frontend-backend-pipelines.png)

---

### **4 — Configure GitHub Webhooks**

We will make Jenkins automatically trigger pipelines on **GitHub push**.

Repeat this for both the repos

1. Open **GitHub → Repository → Settings → Webhooks → Add webhook**.
2. Enter:

   * **Payload URL:**

     ```
     http://<build-server-public-ip>:8080/github-webhook/
     ```
   * **Content type:** `application/json`
   * **Secret:** leave blank (optional)
   * **Which events would you like to trigger this webhook?** → **Just the push event**
3. Click **Add webhook**.

✅ Whenever you push code to the repo, Jenkins will automatically trigger the pipeline.

![Frontend Webhook](artifacts/18.1-frontend-webhook.png)
![Backend Webhook](artifacts/18.2-backend-webhook.png)

---

### **5 — Test Frontend Pipeline**

1. Make a change to **frontend repo** (e.g., edit `frontend_version.json` and change the version).
2. Push to GitHub from **build server**:

```bash
cd ~/ci-cd-workshop/ci-cd-workshop-frontend
```

```bash
git add .
```

```bash
git commit -m "Test frontend update"
```

```bash
git push origin main
```

3. Open Jenkins → **frontend-deploy pipeline** → check **Builds**.
4. The first time it will fail, as the "S3_BUCKET" parameter contains the default value.
5. Edit the Jenkinsfile and update the parameter value to your actual bucket. Change value of **Default Value** from **ci-cd-workshop-frontend-<your-name>** to your bucket name.
6. Update your `index.html` file to point to your backend server’s public IP:
   Find this line:

```js
const API_BASE = "http://127.0.0.1:5000";
```

Replace it with:

```js
const API_BASE = "http://<your-backend-server-ip>:5000";
```

7. Then **add → commit → push** again.
8. Verify pipeline runs successfully and static site is deployed to S3.

![Frontend Jenkinspipeline Success](artifacts/19-frontend-jenkinspipeline-success.png)

---

### **6 — Test Backend Pipeline**

1. Make a change to **backend repo** (e.g., edit `backend_version.txt`).
2. Push to GitHub from **build server**:

```bash
cd ~/ci-cd-workshop/ci-cd-workshop-backend
```

```bash
git add .
```

```bash
git commit -m "Test backend update"
```

```bash
git push origin main
```

3. Open Jenkins → **frontend-deploy pipeline** → check **Builds**.
4. The first time it will fail, as the "BACKEND_SERVER_IP" parameter contains the default value.
5. Edit the Jenkinsfile and update the parameter value to your actual backend-server-ip. Change value of **Default Value** from **<backend-server-ip>** to your backend-server-ip public IP.
5. Also make a small change and follow the above steps to add, commit and push
4. Verify pipeline runs successfully and static site is deployed to S3.

![Backend Jenkinspipeline Success](artifacts/20-backend-jenkinspipeline-success.png)

---

### **Congratulations!**

* Frontend and backend are now fully automated using **Jenkins pipelines**.
* **GitHub webhooks** trigger CI/CD whenever you push new code.
* Backend deploys via **SSH** using the existing EC2 key pair.
* Frontend deploys to **S3 bucket** using the parameterized Jenkinsfile.

---


## **Pending. Cleanup**

Below is **a full cleanup script for ALL your workshop resources**, written exactly in **your variable style**, using **filters by Name**, and **NO paging (`--no-cli-pager`)**, so **you will NEVER be asked to press `q`**.

---

# ✅ **FULL CLEANUP SCRIPT — NO PAGER, NO PROMPTS**

> ⚠️ *This will delete EVERYTHING created for the CI/CD workshop.*

---

## 🚀 **VARIABLES**

```bash
REGION="us-east-1"

# Build server
BUILD_SERVER_NAME="ci-cd-workshop-build-server"
BUILD_SERVER_ROLE="ci-cd-workshop-build-server-role"
BUILD_KEY_PAIR="ci-cd-workshop"

# Frontend S3 bucket
FRONTEND_BUCKET="ci-cd-workshop-frontend-<your-name>"

# Backend server
BACKEND_SERVER_NAME="ci-cd-workshop-backend-server"
BACKEND_SERVER_ROLE="ci-cd-workshop-backend-server-role"

# DynamoDB tables
DDB_ASSIGN_TABLE="ci-cd-workshop-assignments"
DDB_SUBMIT_TABLE="ci-cd-workshop-submissions"

# Build server
BUILD_INSTANCE_PROFILE="ci-cd-workshop-build-server-role"

# Backend server
BACKEND_INSTANCE_PROFILE="ci-cd-workshop-backend-server-role"

# Disable AWS pager
export AWS_PAGER=""
```

---

## ✅ **1. Delete EC2 Instances (Build + Backend)**

### 🔍 **Get Instance IDs**

```bash
BUILD_INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=$BUILD_SERVER_NAME" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text \
    --region $REGION \
    --no-cli-pager)

echo "Build Server Instance: $BUILD_INSTANCE_ID"

BACKEND_INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=$BACKEND_SERVER_NAME" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text \
    --region $REGION \
    --no-cli-pager)

echo "Backend Server Instance: $BACKEND_INSTANCE_ID"
```

---

## 🔐 **2. Get Security Groups from Instances**

```bash
BUILD_SG_IDS=$(aws ec2 describe-instances \
    --instance-ids $BUILD_INSTANCE_ID \
    --query "Reservations[].Instances[].SecurityGroups[].GroupId" \
    --output text \
    --region $REGION \
    --no-cli-pager)

echo "Build Server SGs: $BUILD_SG_IDS"

BACKEND_SG_IDS=$(aws ec2 describe-instances \
    --instance-ids $BACKEND_INSTANCE_ID \
    --query "Reservations[].Instances[].SecurityGroups[].GroupId" \
    --output text \
    --region $REGION \
    --no-cli-pager)

echo "Backend Server SGs: $BACKEND_SG_IDS"
```

---

## 💥 **3. Terminate Instances**

```bash
aws ec2 terminate-instances \
    --instance-ids $BUILD_INSTANCE_ID $BACKEND_INSTANCE_ID \
    --region $REGION \
    --no-cli-pager
```

---

## 🧹 **4. Delete Security Groups**

```bash
for sg in $BUILD_SG_IDS $BACKEND_SG_IDS; do
  echo "Deleting SG: $sg"
  aws ec2 delete-security-group \
      --group-id "$sg" \
      --region $REGION \
      --no-cli-pager
done
```

---

## 🔑 **5. Delete Key Pair**

```bash
aws ec2 delete-key-pair \
    --key-name "$BUILD_KEY_PAIR" \
    --region $REGION \
    --no-cli-pager
```

---

## 🪣 **6. Empty & Delete Frontend S3 Bucket**

```bash
aws s3 rm s3://$FRONTEND_BUCKET --recursive --no-cli-pager

aws s3 rb s3://$FRONTEND_BUCKET --force --no-cli-pager
```

---

## 🍽 **7. Delete DynamoDB Tables**

```bash
aws dynamodb delete-table \
    --table-name $DDB_ASSIGN_TABLE \
    --region $REGION \
    --no-cli-pager

aws dynamodb delete-table \
    --table-name $DDB_SUBMIT_TABLE \
    --region $REGION \
    --no-cli-pager
```
---

## ✅ **CLEANUP: BACKEND ROLE**

### **1. List attached managed policies**

```bash
aws iam list-attached-role-policies \
    --role-name $BACKEND_SERVER_ROLE \
    --no-cli-pager
```

### **2. Detach each policy (run for each ARN shown above)**

For example:

```bash
aws iam detach-role-policy \
    --role-name $BACKEND_SERVER_ROLE \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
    --no-cli-pager

aws iam detach-role-policy \
    --role-name $BACKEND_SERVER_ROLE \
    --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess \
    --no-cli-pager
```

*(These two are the only policies you attached.)*

### **3. Remove role from instance profile**

```bash
aws iam remove-role-from-instance-profile \
    --instance-profile-name $BACKEND_INSTANCE_PROFILE \
    --role-name $BACKEND_SERVER_ROLE \
    --no-cli-pager
```

### **4. Delete instance profile**

```bash
aws iam delete-instance-profile \
    --instance-profile-name $BACKEND_INSTANCE_PROFILE \
    --no-cli-pager
```

### **5. Delete the role**

```bash
aws iam delete-role \
    --role-name $BACKEND_SERVER_ROLE \
    --no-cli-pager
```

---

## ✅ **CLEANUP: BUILD SERVER ROLE**

### **1. List attached managed policies**

```bash
aws iam list-attached-role-policies \
    --role-name $BUILD_SERVER_ROLE \
    --no-cli-pager
```

### **2. Detach policy**

```bash
aws iam detach-role-policy \
    --role-name $BUILD_SERVER_ROLE \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
    --no-cli-pager
```

### **3. Remove role from instance profile**

```bash
aws iam remove-role-from-instance-profile \
    --instance-profile-name $BUILD_INSTANCE_PROFILE \
    --role-name $BUILD_SERVER_ROLE \
    --no-cli-pager
```

### **4. Delete instance profile**

```bash
aws iam delete-instance-profile \
    --instance-profile-name $BUILD_INSTANCE_PROFILE \
    --no-cli-pager
```

### **5. Delete role**

```bash
aws iam delete-role \
    --role-name $BUILD_SERVER_ROLE \
    --no-cli-pager
```

