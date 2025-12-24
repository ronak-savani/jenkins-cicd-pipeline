Deployment Architecture


#What this setup does:

Jenkins Controller - This is my main Jenkins server that runs the pipeline
Git Repository - Where I store my PHP application code (using GitLab in my case)
Build Agent - The Jenkins worker that actually runs the pipeline steps
Target Server - My production server where the application gets deployed
Web Server - Nginx with PHP-FPM that serves the application
Database - MySQL database that the application connects to

How everything flows together:
First, Jenkins pulls the latest code from Git. Then it prepares a new release with a timestamp (like 20231201143000 for Dec 1, 2025 at 2:30 PM). It transfers the files to the server using rsync over SSH. Once files are there, it sets up permissions and configuration. The key part is switching a symbolic link (like a shortcut) to point to the new release directory - this happens instantly so there's no downtime. Finally, it checks if the application is working and sends me an email with the results.


#server looks like after deployment:
/var/www/html/
├── config/           # My configuration files (.env, etc.)
├── releases/         # All my past versions
│   ├── 20251201120000/  # Version from Dec 1, 12:00 PM
├── current@ → releases/20251201120000/  # Points to current live version
└── shared/          # Files that persist between deployments (uploads, logs)