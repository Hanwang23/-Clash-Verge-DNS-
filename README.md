// Clash Verge Rev / Mihomo global extension.
// Keep TUN enabled in the app; app-managed TUN settings must match below.
// Node hostname bootstrap uses encrypted direct DoH. Website DNS uses proxy DoH.
// DNS failure does not fall back to system DNS or plaintext DNS.
function main(config, profileName) {
  const groupName = "DNS-PROXY";
  // Optional preferred node. If unavailable in a subscription, use its first node.
  const preferredNode = "SOCKS5 72.1.178.115:7009";
  const nodes = (config.proxies || []).filter(function (p) {
    return p && p.name && ["direct", "reject", "dns", "loopback", "pass"].indexOf(p.type) < 0;
  }).map(function (p) { return p.name; });
  const providers = Object.keys(config["proxy-providers"] || {});
  if (!nodes.length && !providers.length) {
    throw new Error("DNS guard requires at least one proxy node/provider.");
  }
  const preferredIndex = nodes.indexOf(preferredNode);
  if (preferredIndex > 0) nodes.unshift(nodes.splice(preferredIndex, 1)[0]);
  const group = { name: groupName, type: "select" };
  if (nodes.length) group.proxies = nodes;
  if (providers.length) {
    group.use = providers;
    group["exclude-type"] = "direct|reject|dns|loopback|pass";
  }
  config["proxy-groups"] = (config["proxy-groups"] || []).filter(function (g) {
    return g.name !== groupName;
  }).concat([group]);

  const proxyDoh = [
    "https://1.1.1.1/dns-query#" + groupName,
    "https://1.0.0.1/dns-query#" + groupName
  ];
  const bootstrapDoh = ["https://223.5.5.5/dns-query#DIRECT"];
  // Replace the entire DNS object to remove inherited direct policies/fallbacks.
  config.dns = {
    enable: true,
    listen: "127.0.0.1:1053",
    ipv6: false,
    "enhanced-mode": "fake-ip",
    "fake-ip-range": "198.18.0.1/16",
    "fake-ip-filter-mode": "blacklist",
    "fake-ip-filter": ["*.lan", "*.local", "localhost"],
    "use-hosts": true,
    "use-system-hosts": true,
    "prefer-h3": false,
    "respect-rules": false,
    "default-nameserver": bootstrapDoh,
    "proxy-server-nameserver": bootstrapDoh,
    nameserver: proxyDoh,
    "direct-nameserver": proxyDoh,
    "direct-nameserver-follow-policy": false
  };
  config.tun = Object.assign({}, config.tun || {}, {
    enable: true,
    "auto-route": true,
    "auto-detect-interface": true,
    "strict-route": true,
    "dns-hijack": ["any:53", "tcp://any:53"]
  });
  // Also set IPv6 off in the app; the app owns this top-level setting.
  config.ipv6 = false;
  // Known browser DoH/DoT endpoints use the DNS proxy in rule mode as well.
  // No special routing for leak-test websites: the policy applies to all sites.
  const privacyRules = [
    "DOMAIN,dns.google," + groupName,
    "DOMAIN-SUFFIX,cloudflare-dns.com," + groupName,
    "DOMAIN-SUFFIX,one.one.one.one," + groupName,
    "DOMAIN,dns.quad9.net," + groupName,
    "IP-CIDR,1.1.1.1/32," + groupName + ",no-resolve",
    "IP-CIDR,1.0.0.1/32," + groupName + ",no-resolve",
    "IP-CIDR,8.8.8.8/32," + groupName + ",no-resolve",
    "IP-CIDR,8.8.4.4/32," + groupName + ",no-resolve",
    "IP-CIDR,9.9.9.9/32," + groupName + ",no-resolve",
    "IP-CIDR,149.112.112.112/32," + groupName + ",no-resolve",
    "DST-PORT,853," + groupName
  ];
  config.rules = privacyRules.concat((config.rules || []).filter(function (r) {
    return privacyRules.indexOf(r) < 0;
  }));
  return config;
}
