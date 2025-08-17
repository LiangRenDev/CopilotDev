# Git Multiple Account Management Guide

This guide shows how to manage multiple Git accounts on the same machine.

## Current Setup

### Global Configuration (Xero Work Account)
- user.name: liang-ren_xero  
- user.email: liang.ren@xero.com

### Local Configuration (Personal Account - This Repository)
- user.name: YourPersonalName (UPDATE THIS)
- user.email: your.personal@email.com (UPDATE THIS)

## Quick Setup Steps

### 1. Update Personal Details for This Repository
`powershell
cd "c:\XeroCode\Copilot"
git config --local user.name "Your Real Name"
git config --local user.email "your.real@email.com"
`

### 2. Verify Configuration  
`powershell
git config --local --list
`

### 3. Test with Empty Commit
`powershell
git commit --allow-empty -m "Test personal account"  
git log -1 --pretty=format:"%an <%ae>"
`

### 4. Push to Personal GitHub
`powershell
git push origin master
`

## Method Options

1. **Repository-Specific (Current)**: Each repo has local config
2. **Conditional Config**: Auto-switch based on directory  
3. **SSH Keys**: Different keys for different accounts

Work repositories will continue using your Xero account automatically.
