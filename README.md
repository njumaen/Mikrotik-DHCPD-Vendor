# Mikrotik DHCPD Vendor lease comment
 

This script for your Mikrotik DHCP Server sets the vendor as comment for a new DHCP lease.
It then optionally sends information about the lease by telegram.

``
:local mac [:pick $leaseActMAC 0 8];
:foreach i in=[/ip dhcp-server lease find mac-address=$leaseActMAC] do={
   :if ( ([:len [/ip dhcp-server lease get $i comment]] = 0) and ( [/ip dhcp-server lease get $i dynamic] )  ) do={

      :local mvurl "https://api.macvendors.com/$mac";
      :local result [/tool fetch url=$mvurl as-value output=user ];
      :local vendor ($result->"data");
      :log info "MAC: $mac / VENDOR: $vendor";
      /ip dhcp-server lease set comment="* VENDOR: $vendor" $i

      :local urlstring "DHCP $leaseActMAC - $leaseActIP -  $vendor"
      :local urlEncoded
      :for j from=0 to=([:len $urlstring] - 1) do={ 
          :local char [:pick $urlstring $j]
          :if ($char = " ") do={:set $char "%20"}
          :if ($char = "-") do={:set $char "%2D"}
          :set urlEncoded ($urlEncoded . $char)    
      }
 
     #/tool fetch url="https://api.telegram.org/<yourbotidhere>/sendMessage\?chat_id=<yourchatidhere>&text=$urlEncoded" keep-result=no

   }
}
``