---
layout: post
title: Assigning Azure Active Directory Roles for users and service principals using Python, az rest and Graph API 
date:   2023-03-21 
logo: 'fa fa-code'
comments: true
---
 
I recently got the question on how to assign Azure AD roles to Azure AD `users` and `app registration (service principal (SPN))` using the [Azure AD Graph API] with the [az cli] & [az rest] command.

[Azure AD Graph API]:https://learn.microsoft.com/en-us/graph/use-the-api

[az cli]:https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli

[az rest]:https://learn.microsoft.com/en-us/cli/azure/reference-index?view=azure-cli-latest#az-rest

Here's an example Python script to assign the `Global Administrator`  role to a `user`:

*(replace the **'valid-user-principal-object-id'** with the objectId of the user object)*

``` python
import subprocess
import re

ansi_escape = re.compile(r'\x1B\[[0-?]*[ -/]*[@-~]')

def callTheAPI():
  URI="https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments"

  USER_PRINCIPAL_OBJECT_ID="66bb642f-3dae-4619-b584-1c91ade9aede"
  DIRECTORY_ROLE_TEMPLATE_ID="62e90394-69f5-4237-9190-012177145e10"

  BODY={}
  BODY['principalId']=USER_PRINCIPAL_OBJECT_ID
  BODY['roleDefinitionId']=DIRECTORY_ROLE_TEMPLATE_ID
  BODY['directoryScopeId']="/"

  assignGlobalAdminCommand='az rest --method POST --uri '+URI+' --header Content-Type=application/json --body "'+str(BODY)+'"'

  proc = subprocess.Popen(assignGlobalAdminCommand,cwd=None, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, shell=True)
  while True:
    line = proc.stdout.readline()
    if line:
      thetext=ansi_escape.sub('', line.decode('utf-8').rstrip('\r|\n'))
      print(thetext)
    else:
      break

callTheAPI()
```

A copy of the Python code can be found on my Github page over here: [AADRoleAssignment_User.py
]

[AADRoleAssignment_User.py]:https://github.com/pvyver/AAD-RoleAssignment/blob/main/AADRoleAssignment_User.py

Here's an example Python script to assign the `Global Administrator`  role to an `App Registration (Service Principal (SPN))`:

*(replace the **'valid-app-object-id'** with the objectId of the App registration)*
>It's the object id of the service principal you need, not the application. You can find the service principal under Enterprise Applications in Azure portal's Azure AD blade. In its Properties you'll find the object id.

``` python
import subprocess
import re

ansi_escape = re.compile(r'\x1B\[[0-?]*[ -/]*[@-~]')

def callTheAPI():

  SERVICE_PRINCIPAL_OBJECT_ID="valid-service-principal-object-id"

  DIRECTORY_ROLE_TEMPLATE_ID="62e90394-69f5-4237-9190-012177145e10" # Global Admin Role Template Id

  URI='https://graph.microsoft.com/v1.0/directoryRoles/roleTemplateId='+DIRECTORY_ROLE_TEMPLATE_ID+'/members/$ref' 

  BODY={}
  BODY['@odata.id']='https://graph.microsoft.com/v1.0/directoryObjects/'+SERVICE_PRINCIPAL_OBJECT_ID

  assignGlobalAdminCommand='az rest --method POST --uri '+URI+' --header Content-Type=application/json --body "'+str(BODY)+'"'

  proc = subprocess.Popen(assignGlobalAdminCommand,cwd=None, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, shell=True)
  while True:
    line = proc.stdout.readline()
    if line:
      thetext=ansi_escape.sub('', line.decode('utf-8').rstrip('\r|\n'))
      print(thetext)
    else:
      break

callTheAPI()
```

A copy of the Python code can be found on my Github page over here: [AADRoleAssignment_App.py]

[AADRoleAssignment_App.py]:https://github.com/pvyver/AAD-RoleAssignment/blob/main/AADRoleAssignment_App.py
