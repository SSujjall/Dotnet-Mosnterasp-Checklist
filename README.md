# 🧾 Dotnet-Monsterasp-Checklist

A quick checklist to ensure smooth **.NET backend deployment** via **Web Deploy** to [MonsterASP](https://www.monsterasp.net/).

---

## ✅ MonsterASP Deployment Checklist

### 🔒 SSL & HTTPS
- HTTPS **must be enabled** using a valid **SSL certificate**.  
- **HTTPS redirection must be turned off** in Monsterasp's Https Config.

---

### ⚙️ Web Deploy Configuration

Set the following credentials in your **GitHub Secrets** or environment variables:

| Key | Description | Example |
|-----|--------------|----------|
| `SERVERNAME` | Web Deploy server URL | `https://site****.siteasp.net:8172` |
| `USERNAME` | MonsterASP deployment username | `site****` |
| `WEB_SITE_NAME` | Website name (same as username in most cases) | `site****` |
| `PASSWORD` | Deployment password provided by MonsterASP | `********` |

---

### 🚀 Additional Notes
- Ensure the **Web Deploy service** is active in MonsterASP hosting.
- Test deployment manually using Visual Studio before automating via GitHub Actions.
- Verify that your `.sln` or `.csproj` path is correct inside the workflow.
- AppSettings in `appsettings.Production.json` should match your server setup.

---

## ⚡ GitHub Actions Example (Web Deploy to MonsterASP)

Add this file as `.github/workflows/deploy.yml` in your repository:

```yaml
name: Test Backend CI/CD Pipeline

on:
  push:
    branches:
      - main # Trigger on push to this branch

jobs:
  build_and_deploy_backend:
    runs-on: windows-latest

    steps:
      # Checkout the code from the repository
      - name: Checkout repository
        uses: actions/checkout@v4

      # Set up .NET SDK
      - name: Setup .NET 9
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x

      # Install EF Core Tools
      - name: Install EF Core Tools 9.0.9
        run: dotnet tool install --global dotnet-ef --version 9.0.9

      # Verify dotnet-ef version
      - name: Verify dotnet-ef version
        run: dotnet ef --version

      # Restore dependencies
      - name: Install dependencies
        run: dotnet restore Test.Backend/Test.Backend.sln

      # Build the project
      - name: Build
        run: dotnet build Test.Backend/Test.Backend.sln --configuration Release --no-restore

      # Publish the project
      - name: Publish
        run: dotnet publish Test.Backend/Test.Backend.sln --configuration Release --output ./publish --runtime win-x86

      # Debug: List contents of the publish folder to verify the files are there
      - name: List published files
        run: dir ./publish

      # Handle appsettings.json based on environment variable
      - name: Handle appsettings.json
        run: |
          # If SKIP_APPSETTINGS is true, remove appsettings.json from deployment
          if ("${{ vars.SKIP_APPSETTINGS }}" -eq "true") {
              Write-Host "SKIP_APPSETTINGS is true. Removing appsettings.json..."
              Remove-Item -Path ./publish/appsettings.json -Force -ErrorAction SilentlyContinue
              Write-Host "appsettings.json removed from deployment package."
          } else {
              Write-Host "Including appsettings.json in deployment."
          }
        shell: pwsh

      # Deploy to MonsterASP.NET via WebDeploy
      - name: Deploy to MonsterASP.NET via WebDeploy
        uses: rasmusbuchholdt/simply-web-deploy@2.1.0 # check github repo for latest version
        with:
          website-name: ${{ secrets.MONSTERASP_WEB_WEBSITENAME }}  # Use WebDeploy Access site as website name
          server-computer-name: ${{ secrets.MONSTERASP_WEB_SERVERNAME }}  # Use WebDeploy Access server/host name as WebDeploy server along with port eg: https://siteXXXX.siteasp.net:8172
          server-username: ${{ secrets.MONSTERASP_WEB_USERNAME }}  # Use WebDeploy Access login as WebDeploy username
          server-password: ${{ secrets.MONSTERASP_WEB_PASSWORD }}  # Use WebDeploy Access password as password
          source-path: publish/  # Folder containing the published files
          target-path: /  # Deploy to the root of the website

      # Apply database migrations
      - name: Apply database migrations
        run: dotnet ef database update --configuration Release --no-build --project ./Test.Data/Test.Data.csproj
        env:
          ASPNETCORE_ENVIRONMENT: Production
          ConnectionStrings__BlogDB: ${{ secrets.MONSTERASP_DB_CONNECTION_STRING }}
