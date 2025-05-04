# 🚀 CI/CD Deployment from GitHub to AWS EC2 / Digital Ocean

```bash
        mac
    (pub, priv)
        ⬇️
  deploying machine
      (pub)
        ⬆️
      github
      (priv)
```

This guide documents the full process to set up continuous deployment from a GitHub repo to an AWS EC2 instance using GitHub Actions.

---

## 🖥️ 1. Launch EC2 Instance

- Go to AWS EC2 console.
- Launch an Ubuntu server (e.g., Ubuntu 22.04 LTS).
- Select or create a new key pair. Download the `.pem` file (e.g., `adt-test-1.pem`).
- Allow inbound SSH (port 22), HTTP (80), and HTTPS (443) in security group.

---

## (follow this only if u didnt download .pem - you will need to create it otherwise)

## 🔐 2. Set Up SSH Access from Local Machine and run EC2 Instance

### Generate SSH Key Pair (locally):

```bash
ssh-keygen -t ed25519 -C "adt-test-1"
or
ssh-keygen -t rsa -C "adt-test-1"
# Output: ~/.ssh/adt-test-1 and adt-test-1.pub
```

## change permissions

```bash
chmod 700 adt-test-1
```

### Start EC2 with private key :

(if created)

```bash
 ssh -i ~/.ssh/adt-test-1.pem ubuntu@<EC2_PUBLIC_IP>
```

(if downloaded)

```bash
ssh -i adt-test-1 ubuntu@<EC2_PUBLIC_IP>
```

## 🛠️ 3. Set Up Node.js App on EC2

```bash
sudo apt update
sudo apt install nodejs
sudo apt install npm
# Setup node js from harkirat's ppt
https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-20-04

sudo npm i -g pm2
```

---

## 📁 4. Clone Your GitHub Repo on EC2

(app name can be anything)

```bash
git clone <github_url>
cd <your-repo>
npm install
npm run build

npm run start
or
pm2 start npm --name "app name" -- start

pm2 startup
pm2 save
```

---

## 🔐 5. Give EC2 Access to Your GitHub Repo

### On EC2:

(adt-test-1 can be any name otherwise like github-ec2)

```bash
ssh-keygen -t rsa -C "adt-test-1"
cat ~/.ssh/id_adt-test-1.pub
```

Copy the output and add to GitHub:

- Go to GitHub → Settings → SSH and GPG Keys → **New SSH Key**

### Test SSH Access from EC2:

```bash
ssh -T git@github.com
# Should say: Hi <username>! You've successfully authenticated...
```

if not

```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts
```

if not

```bash
nano ~/.ssh/config
```

write in file:

```bash
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_adt-test-1
```

## chnage permissions

```bash
chmod 700 ~/.ssh/id_adt-test-1
chmod 700 ~/.ssh/id_adt-test-1.pub
chmod 700 ~/.ssh
```

---

## ⚙️ 6. GitHub Actions CI/CD Setup

### Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up SSH agent with EC2 key
        uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.EC2_SSH_KEY }}

      - name: Add EC2 instance to known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.EC2_IP }} >> ~/.ssh/known_hosts

      - name: Deploy and restart on EC2
        run: |
          ssh ubuntu@${{ secrets.EC2_IP }} << 'EOF'
            cd ~/ci-cd-test
            git pull origin main
            npm install --legacy-peer-deps
            npm run build
            pm2 restart "app name" || pm2 start npm --name "app name" -- start
          EOF
```

---

## 🔒 7. Add GitHub Secrets

Go to your repo → **Settings → Secrets and variables → Actions** → **New repository secret**

| Secret Key    | Value                                 |
| ------------- | ------------------------------------- |
| `EC2_SSH_KEY` | Content of `~/.ssh/adt-test-1`        |
| `EC2_IP`      | Your EC2 instance's public IP address |

---

## move your public keys in ~/.ssh/authorized_keys of EC2

adt-test-1.pub

```bash
mkdir -p ~/.ssh
echo "ssh-rsa AAAAB3... adt-test-1" >> ~/.ssh/authorized_keys
cat ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

## ✅ 8. Done! Push and Watch It Deploy

(push not necessarily on EC2 local, any local cloned repo will get pushed into EC2 instance only)

```bash
git add .
git commit -m "Trigger deployment"
git push origin main
```

Then go to GitHub → **Actions Tab** → You’ll see the deployment job run automatically 🚀
