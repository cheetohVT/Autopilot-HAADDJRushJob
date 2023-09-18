# Autopilot-HAADDJRushJob
Solution to aid in faster Autopilot Hybrid Join, by driving computer object creation in AAD as a result of domain controller audit events.

.Synopsis

Use an event-driven approach to kick off hybrid join process

.Principles

1. Use auditing to generate a security event when the 'userCertificate' property of a computer account is added.
2. Use a source-initiated event log subscription to distribute events to the ADConnect server
3. Consume auditing events to drive ADConnect Sync cycles.

.Operations

1. Configure auditing:

   a. Configure the SACL.  On the OU or OUs that will see Autopilot Device creation, configure the SACL to audit 'write' for descendant Computer Objects' 'userCertificate' property.  This will cause event 5136 to be generated whenever the userCertificate property of a computer object is populated
   
   b. Configure domain controller computers Adavance Audit Policy Configuration.  DSAccess\Audit Directory Service Changes:

      SUCCESS = ENABLE.

      This enableds DCs to audit on all DS Access changes
   
3. Configure Event log collection

   a. Configure Group policy for event source conmputers:
   
         Computer Configuration / Administrative Templates / Windows Conmponents / Configure target subscription Manager/ Enabled: "Server=http://<Server_FQDN>:5985/wsman/Subscription/WEC,Refresh=10"
   
   b. Configure Winrm on sources:

         winrm qc -q
   
   c. Configure permissions for NETWORK SERVICE to read Security events on DCs (sources):

        wevtutil sl security /ca:O:BAG:SYD:(A;;0xf0005;;;SY)(A;;0x5;;;BA)(A;;0x1;;;S-1-5-32-573)(A;;0x1;;;S-1-5-20)
   
   d. configure winrm and event collection on the ADConnect server:
      
       winrm qc -q
       wecutil qc /q
   
   d. Configure WinRM permsission ont he collector (2016+):
   
       netsh http delete urlacl url=http://+:5985/wsman/
       netsh http add urlacl url=http://+:5985/wsman/ sddl=D:(A;;GX;;;S-1-5-80-569256582-2953403351-2909559716-1301513147-412116970)(A;;GX;;;S-1-5-80-4059739203-877974739-1245631912-527174227-2996563517)
       netsh http delete urlacl url=https://+:5986/wsman/
       netsh http add urlacl url=https://+:5986/wsman/ sddl=D:(A;;GX;;;S-1-5-80-569256582-2953403351-2909559716-1301513147-412116970)(A;;GX;;;S-1-5-80-4059739203-877974739-1245631912-527174227-2996563517)
   
4. Add a subscription on the ADConnect server:

          wecutil cs subscription.xml

5. Add task on the ADConnect Box to run a delta sycn when the event is received:
   
          schtasks /xml ScheduledTask.xml /TN "cheetohVT/Start AD Sync Cycle"


Additional thoughts

If the ADConnect server is in a different site than where the userCertificate change occurred, enable change notify for the IP Site connect between the AD Sites.

This solution results in quick Hybrid-joined Computer object creation in AAD, saving time during AUtopilot deployments.



