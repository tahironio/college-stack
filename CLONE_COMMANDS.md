# Clone commands for server setup

**⚠️ SECURITY: Do NOT commit this file. It will be deleted after use.**

Save your PAT as a variable, then clone all repos:

```bash
cd /opt/college
export GITHUB_PAT="YOUR_TOKEN_HERE"
git clone https://$GITHUB_PAT@github.com/tahironio/ais-college.git AIS/ais-college
git clone https://$GITHUB_PAT@github.com/tahironio/lampa-lms.git distantes
git clone https://$GITHUB_PAT@github.com/tahironio/schedule-app.git Shedule
unset GITHUB_PAT
ls -la
```

**After cloning, revoke this PAT on GitHub immediately.**
