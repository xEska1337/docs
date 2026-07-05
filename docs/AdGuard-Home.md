AdGuard Home is a free and open source, powerful network-wide ads & trackers blocking DNS server. 

## Architecture Overview

This network environment utilizes **OPNsense** as the primary firewall and routing platform.

* **Primary DNS:** An AdGuard Home instance running directly on OPNsense, installed via the `os-adguardhome-maxit` plugin.
* **Secondary DNS:** A secondary AdGuard Home instance deployed as a Docker container on **TrueNAS SCALE** to provide redundancy.
* **Synchronization:** An `adguardhome-sync` Docker container is used to automatically replicate configuration changes from the primary instance to the secondary instance.

---

## Upstream DNS Configuration

To ensure privacy and security, DNS queries are routed using DNS-over-HTTPS (DoH). The following upstream DNS servers are configured:

```text
https://dns.quad9.net/dns-query
https://security.cloudflare-dns.com/dns-query

```

---

## Blocklists

The instances are configured to use the following blocklists for DNS filtering:

* **AdGuard DNS Filter:** `https://adguardteam.github.io/HostlistsRegistry/assets/filter_1.txt`
* **WindowsSpyBlocker - Hosts spy rules** `https://adguardteam.github.io/HostlistsRegistry/assets/filter_23.txt`
* **POL: CERT Polska List of malicious domains:** `https://adguardteam.github.io/HostlistsRegistry/assets/filter_41.txt`
* **Steven Black's List:** `https://adguardteam.github.io/HostlistsRegistry/assets/filter_33.txt`
* **POL: Polish filters for Pi-hole:** `https://adguardteam.github.io/HostlistsRegistry/assets/filter_14.txt`
* **NoCoin Filter List:** `https://adguardteam.github.io/HostlistsRegistry/assets/filter_8.txt`
* **EasyList:** `https://easylist-downloads.adblockplus.org/easylist_noelemhide.txt`
* **EasyList_Polish:** `https://raw.githubusercontent.com/MajkiIT/polish-ads-filter/master/polish-adblock-filters/adblock.txt`
* **Phishing:** `https://urlhaus.abuse.ch/downloads/hostfile/`
* **PolskiAntyirytujacyDodatekSpecjalny:** `https://raw.githubusercontent.com/FiltersHeroes/PolishAnnoyanceFilters/master/PPB.txt`
* **Polskie_Filtry_Anty_Adblockowe:** `https://raw.githubusercontent.com/olegwukr/polish-privacy-filters/master/anti-adblock.txt`
* **Malware:** `https://v.firebog.net/hosts/RPiList-Malware.txt`
* **Phishing:** `https://v.firebog.net/hosts/RPiList-Phishing.txt`
* **simple_ad:** `https://s3.amazonaws.com/lists.disconnect.me/simple_ad.txt`
* **notrack-malware:** `https://gitlab.com/quidsup/notrack-blocklists/raw/master/notrack-malware.txt`
* **phishing_army_blocklist_extended:** `https://phishing.army/download/phishing_army_blocklist_extended.txt`
* **simple_malvertising:** `https://s3.amazonaws.com/lists.disconnect.me/simple_malvertising.txt`
* **AntiMalwareHosts:** `https://raw.githubusercontent.com/DandelionSprout/adfilt/master/Alternate%20versions%20Anti-Malware%20List/AntiMalwareHosts.txt`
* **Tracking & Telemetry Lists:** `https://v.firebog.net/hosts/Prigent-Ads.txt`
* **Spam:** `https://raw.githubusercontent.com/FadeMind/hosts.extras/master/add.Spam/hosts`
* **KADhosts:** `https://raw.githubusercontent.com/PolishFiltersTeam/KADhosts/master/KADhosts.txt`
* **KADPrzekrety:** `https://raw.githubusercontent.com/FiltersHeroes/KAD/master/KAD.txt`
* **AdGuardMobileAdsFilter:** `https://filters.adtidy.org/extension/ublock/filters/11.txt`
* **AduardAnnoyancesfilter:** `https://filters.adtidy.org/extension/ublock/filters/14.txt`
* **AdGuard_Tracking_Protection_filter:** `https://filters.adtidy.org/extension/ublock/filters/3.txt`
* **I_dont_care_about_cookies:** `https://www.i-dont-care-about-cookies.eu/abp/`
* **Firebog_Trackers:** `https://v.firebog.net/hosts/Easyprivacy.txt`
* **Firebog_Advertising:** `https://v.firebog.net/hosts/AdguardDNS.txt`
* **Fake - Protects against internet scams, traps & fakes!:** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/fake.txt`
* **Pop-Up Ads - Protects against annoying and malicious pop-up ads!:** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/popupads.txt`
* **Threat Intelligence Feeds - Increases security significantly! (Recommended):** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.txt`
* **Gambling - Protects against gambling content!:** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/gambling.txt`
* **Native Tracker - Broadband tracker of devices, services and operating systems (Xiaomi):** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/native.xiaomi.txt`
* **Native Tracker - Broadband tracker of devices, services and operating systems (Samsung):** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/native.samsung.txt`
* **Native Tracker - Broadband tracker of devices, services and operating systems (Microsoft):** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/native.winoffice.txt`
* **Native Tracker - Broadband tracker of devices, services and operating systems (TikTok (Fingerprinting) Aggressive):** `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/native.tiktok.extended.txt`

---

## Secondary Instance Deployment (TrueNAS)

The secondary AdGuard Home instance is hosted on TrueNAS. Below is the Docker Compose configuration used to spin up the container:

```yaml title="docker-compose.yaml"
services:
  adguardhome:
    image: adguard/adguardhome
    container_name: adguardhome
    restart: unless-stopped
    volumes:
      - /path/workdir:/opt/adguardhome/work
      - /path/confdir:/opt/adguardhome/conf
    ports:
      - 53:53/tcp
      - 53:53/udp
      - 3000:3000
```

---

## AdGuard Home Sync Deployment

To maintain consistency across both DNS servers, `adguardhome-sync` is deployed. Below is the Docker Compose file used for the synchronization service:

```yaml title="docker-compose.yaml"
services:
  adguardhome-sync:
    image: quay.io/bakito/adguardhome-sync
    container_name: adguardhome-sync
    command: run
    environment:
      - ORIGIN_URL=http://
      - ORIGIN_USERNAME=<USERNAME>
      - ORIGIN_PASSWORD=<PASSWORD>
      - REPLICA_URL=http://
      - REPLICA_USERNAME=<USERNAME>
      - REPLICA_PASSWORD=<PASSWORD>
      - CRON=*/30 * * * * # run every 30 minutes
      - RUNONSTART=true
      - LOG_LEVEL=debug
    ports:
      - 8080:8080
    restart: unless-stopped

```