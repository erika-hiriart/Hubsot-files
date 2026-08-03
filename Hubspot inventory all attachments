// hubspot-inventory-all-attachments.js
//
// Finds every PDF/DOC/DOCX file attached anywhere in HubSpot -- across Notes, Emails,
// Calls, Meetings, and Tasks -- and traces each one back to whatever Deal/Company/
// Contact/Ticket it's associated with.
//
// WHY ALL FIVE ENGAGEMENT TYPES: HubSpot does not store attachments directly on a Deal
// or Company record. Every attachment lives on an "engagement" (Note, Email, Call,
// Meeting, or Task), and that engagement is separately associated with whatever CRM
// record you see it on. So "find every attachment on a deal" really means "find every
// engagement associated with that deal, then check each engagement for attachments."
//
// USAGE:
//   HUBSPOT_TOKEN=pat-xxxxxxxx node hubspot-inventory-all-attachments.js
//
// REQUIRED PRIVATE APP SCOPES:
//   crm.objects.contacts.read, crm.objects.companies.read, crm.objects.deals.read,
//   crm.objects.tickets.read (for the association lookups)
//   files.read
//   files.ui_hidden.read   <-- required to fetch attachment files by ID (they're hidden files)
//
// KNOWN UNCERTAINTIES -- please verify against your portal before trusting the output:
//   1. hs_attachment_ids delimiter: this script splits on ";". If a note/email/etc. with
//      multiple attachments comes back looking like one giant ID, log the raw value and
//      adjust SPLIT_DELIMITER below.
//   2. v4 associations batch/read request size: this script chunks at 100 IDs per batch
//      as a conservative default. If you get errors, reduce CHUNK_SIZE; if it's fine,
//      you can likely push it higher and cut down on total calls.
//   3. There is NO batch-read endpoint for the Files API itself, so file metadata is
//      still fetched one call per unique file ID -- that part can't be batched no matter
//      how many engagement types we sweep.

const TOKEN = process.env.HUBSPOT_TOKEN;
if (!TOKEN) {
  console.error('Set HUBSPOT_TOKEN env var to a HubSpot Private App access token.');
  process.exit(1);
}

const BASE = 'https://api.hubapi.com';
const HEADERS = {
  Authorization: `Bearer ${TOKEN}`,
  'Content-Type': 'application/json',
};

const ENGAGEMENT_TYPES = ['notes', 'emails', 'calls', 'meetings', 'tasks'];
const ASSOCIATION_TARGETS = ['deals', 'companies', 'contacts', 'tickets'];
const TARGET_EXTENSIONS = new Set(['pdf', 'doc', 'docx']);
const SPLIT_DELIMITER = ';';
const CHUNK_SIZE = 100;
const FILE_FETCH_CONCURRENCY = 5;

async function hsFetch(path, options = {}) {
  const res = await fetch(`${BASE}${path}`, { ...options, headers: HEADERS });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`${options.method || 'GET'} ${path} failed: ${res.status} ${body}`);
  }
  return res.json();
}

function chunk(arr, size) {
  const out = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}

async function mapWithConcurrency(items, limit, fn) {
  const results = new Array(items.length);
  let idx = 0;
  async function worker() {
    while (idx < items.length) {
      const current = idx++;
      results[current] = await fn(items[current]);
    }
  }
  await Promise.all(Array.from({ length: limit }, worker));
  return results;
}

// Step 1: for a given engagement type, page through everything with an attachment
async function getEngagementsWithAttachments(engagementType) {
  const items = [];
  let after;
  do {
    const body = {
      filterGroups: [
        { filters: [{ propertyName: 'hs_attachment_ids', operator: 'HAS_PROPERTY' }] },
      ],
      properties: ['hs_attachment_ids', 'hs_createdate'],
      limit: 100,
      ...(after ? { after } : {}),
    };
    const page = await hsFetch(`/crm/v3/objects/${engagementType}/search`, {
      method: 'POST',
      body: JSON.stringify(body),
    });
    items.push(...page.results);
    after = page.paging?.next?.after;
  } while (after);
  return items;
}

// Step 2: batch-fetch associations from a set of engagement IDs to a target object type
async function getAssociationsBatch(fromType, toType, ids) {
  const map = new Map(); // engagementId -> [toObjectId, ...]
  for (const idChunk of chunk(ids, CHUNK_SIZE)) {
    try {
      const res = await hsFetch(`/crm/v4/associations/${fromType}/${toType}/batch/read`, {
        method: 'POST',
        body: JSON.stringify({ inputs: idChunk.map((id) => ({ id })) }),
      });
      for (const result of res.results || []) {
        const fromId = result.from.id;
        const toIds = (result.to || []).map((t) => t.toObjectId);
        if (toIds.length) map.set(fromId, [...(map.get(fromId) || []), ...toIds]);
      }
    } catch (e) {
      console.warn(`Association lookup ${fromType}->${toType} failed for a chunk: ${e.message}`);
    }
  }
  return map;
}

async function getFileMetadata(fileId) {
  try {
    return await hsFetch(`/files/v3/files/${fileId}`);
  } catch {
    return null;
  }
}

async function main() {
  // --- Gather all engagements with attachments, across all 5 types ---
  const allEngagements = []; // { type, id, attachmentIds: [...] }
  for (const type of ENGAGEMENT_TYPES) {
    console.log(`Fetching ${type} with attachments...`);
    const results = await getEngagementsWithAttachments(type);
    console.log(`  found ${results.length} ${type} with attachments.`);
    for (const r of results) {
      const ids = (r.properties.hs_attachment_ids || '')
        .split(SPLIT_DELIMITER)
        .map((s) => s.trim())
        .filter(Boolean);
      if (ids.length) allEngagements.push({ type, id: r.id, attachmentIds: ids });
    }
  }
  console.log(`Total engagements with attachments across all types: ${allEngagements.length}`);

  // --- Resolve associations per engagement type -> each CRM target ---
  // associationMap[type][target] = Map(engagementId -> [recordIds])
  const associationMap = {};
  for (const type of ENGAGEMENT_TYPES) {
    const idsOfType = allEngagements.filter((e) => e.type === type).map((e) => e.id);
    if (!idsOfType.length) continue;
    associationMap[type] = {};
    for (const target of ASSOCIATION_TARGETS) {
      console.log(`Resolving ${type} -> ${target} associations...`);
      associationMap[type][target] = await getAssociationsBatch(type, target, idsOfType);
    }
  }

  // --- Build file -> engagement(s) -> record(s) map ---
  const fileToRefs = new Map(); // fileId -> [{ engagementType, engagementId, records: {deals:[],companies:[],contacts:[],tickets:[]} }]
  for (const eng of allEngagements) {
    const records = {};
    for (const target of ASSOCIATION_TARGETS) {
      records[target] = associationMap[eng.type][target].get(eng.id) || [];
    }
    for (const fileId of eng.attachmentIds) {
      if (!fileToRefs.has(fileId)) fileToRefs.set(fileId, []);
      fileToRefs.get(fileId).push({ engagementType: eng.type, engagementId: eng.id, records });
    }
  }

  const uniqueFileIds = [...fileToRefs.keys()];
  console.log(`${uniqueFileIds.length} unique attachment file IDs across all engagement types. Fetching metadata...`);

  const fileResults = await mapWithConcurrency(uniqueFileIds, FILE_FETCH_CONCURRENCY, getFileMetadata);

  const rows = [];
  fileResults.forEach((file, i) => {
    if (!file) return;
    const ext = (file.extension || '').toLowerCase();
    if (!TARGET_EXTENSIONS.has(ext)) return;
    const fileId = uniqueFileIds[i];
    const refs = fileToRefs.get(fileId);
    for (const ref of refs) {
      rows.push({
        fileId,
        name: file.name || '',
        extension: ext,
        url: file.url || '',
        size: file.size || '',
        engagementType: ref.engagementType,
        engagementId: ref.engagementId,
        deals: ref.records.deals.join('|'),
        companies: ref.records.companies.join('|'),
        contacts: ref.records.contacts.join('|'),
        tickets: ref.records.tickets.join('|'),
      });
    }
  });

  console.log(`Matched ${rows.length} PDF/DOC/DOCX file+record rows.`);

  const header = 'fileId,name,extension,url,size,engagementType,engagementId,deals,companies,contacts,tickets\n';
  const csv =
    header +
    rows
      .map((r) =>
        [
          r.fileId,
          `"${r.name.replace(/"/g, '""')}"`,
          r.extension,
          r.url,
          r.size,
          r.engagementType,
          r.engagementId,
          r.deals,
          r.companies,
          r.contacts,
          r.tickets,
        ].join(',')
      )
      .join('\n');

  require('fs').writeFileSync('hubspot-all-attachments.csv', csv);
  console.log('Wrote hubspot-all-attachments.csv');
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
