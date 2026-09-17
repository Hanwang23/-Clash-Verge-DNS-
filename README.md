# -Clash-Verge-DNS-将其粘贴进全局拓展管理即可生效

const domesticNameservers = [
  "https://223.5.5.5/dns-query"
];

const foreignNameservers = [
  "https://1.1.1.1/dns-query",
  "https://8.8.4.4/dns-query"
];

const dnsConfig = {
  "enable": true,
  "listen": "0.0.0.0:1053",
  "ipv6": false,
  "respect-rules": true,
  "use-system-hosts": false,
  "enhanced-mode": "fake-ip",
  "fake-ip-range": "198.18.0.1/16",
  "fake-ip-filter": ["+.lan", "+.local"],
  "default-nameserver": ["1.1.1.1", "8.8.4.4"],
  "nameserver": [...foreignNameservers],
  "proxy-server-nameserver": [...domesticNameservers]
};

// ⭐ TUN 配置 — 关键：劫持系统所有流量包括 DNS
const tunConfig = {
  "enable": true,
  "stack": "mixed",           // 推荐 mixed 模式，兼容性最好
  "dns-hijack": [
    "any:53",                 // 劫持所有发往 53 端口的 DNS 请求
    "tcp://any:53"
  ],
  "auto-route": true,         // 自动设置路由表
  "auto-detect-interface": true,
  "strict-route": true        // 严格路由，防止 DNS 绕过
};

function main(config) {
  const proxyCount = config?.proxies?.length ?? 0;
  const proxyProviderCount =
    typeof config?.["proxy-providers"] === "object"
      ? Object.keys(config["proxy-providers"]).length : 0;
  if (proxyCount === 0 && proxyProviderCount === 0) {
    throw new Error("配置文件中未找到任何代理");
  }

  config["dns"] = dnsConfig;
  config["tun"] = tunConfig;

  return config;
}
