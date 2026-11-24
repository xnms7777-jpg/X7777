import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";

const TARGETS: Record<string, string> = {
  current: process.cwd(),
  desktop: path.join(os.homedir(), "Desktop"),
  downloads: path.join(os.homedir(), "Downloads"),
};

const EXPORTS: Record<string, string> = {
  instagram: "",
  twitter: "",
};

const ACCOUNTS: Record<string, string> = {
  instagram: "mshahrani.78",
  twitter: "",
};

function normalizeAccount(raw: string): string {
  return raw.trim().replace(/^@+/, "");
}

async function readMessagesFromExport(
  source: string,
  count: number,
  account: string
) {
  const baseExportPath = EXPORTS[source] || source;
  const cleanAccount = normalizeAccount(account);

  // إذا كان هناك مجلد فرعي باسم الحساب داخل مسار التصدير ويوجد، يستخدمه؛ وإلا يستخدم المسار الأساسي
  const accountSubPath = cleanAccount
    ? path.join(baseExportPath, cleanAccount)
    : "";
  let exportPath = baseExportPath;

  if (accountSubPath) {
    const exists = await fs
      .stat(accountSubPath)
      .then(() => true)
      .catch(() => false);
    if (exists) {
      exportPath = accountSubPath;
    }
  }

  const stat = await fs.stat(exportPath);
  let messages: any[] = [];

  if (stat.isDirectory()) {
    const entries = await fs.readdir(exportPath);
    for (const name of entries) {
      const full = path.join(exportPath, name);
      const s = await fs.stat(full);
      if (s.isFile() && name.endsWith(".json")) {
        const raw = await fs.readFile(full, "utf8");
        const parsed = JSON.parse(raw);
        if (Array.isArray(parsed)) {
          messages.push(...parsed);
        } else if (Array.isArray(parsed?.messages)) {
          messages.push(...parsed.messages);
        } else {
          messages.push(parsed);
        }
      }
    }
  } else {
    const raw = await fs.readFile(exportPath, "utf8");
    const parsed = JSON.parse(raw);
    if (Array.isArray(parsed)) {
      messages = parsed;
    } else if (Array.isArray(parsed?.messages)) {
      messages = parsed.messages;
    } else {
      messages = [parsed];
    }
  }

  return messages.slice(0, count);
}

export default function register(api: any) {
  api.registerCommand({
    name: "import",
    description: "استيراد رسائل حسب المنصة والعدد والـ Target والحساب",
    handler: async (ctx: any) => {
      const [
        sourceRaw,
        countRaw = "5",
        target = "current",
        accountRaw = "",
      ] = String(ctx?.args || "")
        .trim()
        .split(/\s+/);

      const source = (sourceRaw || "").toLowerCase();
      const count = Number(countRaw) || 5;
      const account = normalizeAccount(accountRaw || ACCOUNTS[source] || "");

      if (!source) {
        return {
          text: "اكتب: /import [instagram|twitter|مسار] [العدد] [target] [account]",
        };
      }

      const messages = await readMessagesFromExport(source, count, account);
      const destDir = TARGETS[target] || target;
      const baseName = EXPORTS[source]
        ? source
        : path.basename(source, path.extname(source));
      const accountSuffix = account ? `-${account}` : "";
      const outFile = path.join(
        destDir,
        `${baseName}${accountSuffix}-imported.json`
      );

      await fs.mkdir(destDir, { recursive: true });
      await fs.writeFile(outFile, JSON.stringify(messages, null, 2), "utf8");

      return {
        text: `تم استيراد ${messages.length} رسائل${account ? ` للحساب ${account}` : ""} إلى ${outFile}`,
      };
    },
  });
}
