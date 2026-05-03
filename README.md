# Mikrotik DHCPD Vendor lease comment
 

This script for your Mikrotik DHCP Server sets the vendor as comment for a new DHCP lease.
It then optionally sends information about the lease by telegram.

```
# DHCP Lease Script – RouterOS 7
# Einbinden: /ip dhcp-server set <name> lease-script=<scriptname>
# Trigger:   add, bound

## Nur bei neuen dynamischen Leases ohne Kommentar ausführen
:if ($leaseServerName = "") do={ :error "kein Server-Kontext" }

:local mac [:pick $leaseActMAC 0 8]
:local leaseid

:foreach i in=[/ip dhcp-server lease find mac-address=$leaseActMAC] do={

  # Nur dynamische Leases ohne bestehenden Kommentar bearbeiten
  :if (
    ([:len [/ip dhcp-server lease get $i comment]] = 0) and
    ([/ip dhcp-server lease get $i dynamic])
  ) do={

    # MAC-Vendor-Abfrage (check-certificate=no für ROS 7 Kompatibilität)
    :local mvurl ("https://api.macvendors.com/" . $mac)
    :local vendor "unbekannt"

    :do {
      :local result [/tool fetch \
        url=$mvurl \
        as-value \
        output=user \
        check-certificate=no \
        http-method=get \
        duration=5s]
      :set vendor ($result->"data")
    } on-error={
      :log warning ("MAC-Vendor-Lookup fehlgeschlagen fuer: " . $mac)
      :set vendor "lookup-fehler"
    }

    :log info ("DHCP Lease: MAC=" . $leaseActMAC . " IP=" . $leaseActIP . " VENDOR=" . $vendor)

    # Kommentar am Lease setzen
    /ip dhcp-server lease set comment=("* VENDOR: " . $vendor) $i

    # URL-Encoding für Telegram-Nachricht
    :local urlstring ("DHCP " . $leaseActMAC . " - " . $leaseActIP . " - " . $vendor)
    :local urlEncoded ""

    :for j from=0 to=([:len $urlstring] - 1) do={
      :local char [:pick $urlstring $j]
      :if ($char = " ") do={ :set char "%20" }
      :if ($char = ":") do={ :set char "%3A" }
      :if ($char = "/") do={ :set char "%2F" }
      :if ($char = "+") do={ :set char "%2B" }
      :if ($char = "=") do={ :set char "%3D" }
      :set urlEncoded ($urlEncoded . $char)
    }

    # Telegram-Benachrichtigung (Bot-ID und Chat-ID eintragen)
    #:local botid     "<DEINE-BOT-ID>"
    #:local chatid    "<DEINE-CHAT-ID>"
    #:local tgurl     ("https://api.telegram.org/bot" . $botid . "/sendMessage?chat_id=" . $chatid . "&text=" . $urlEncoded)

    #:do {
    #  /tool fetch url=$tgurl keep-result=no check-certificate=no http-method=get duration=5s
    #} on-error={
    #  :log warning "Telegram-Benachrichtigung fehlgeschlagen"
    #}

  }
}
```
