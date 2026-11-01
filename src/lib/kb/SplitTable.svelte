<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per cause in the ticket, each with the ticket it was split into.
  // `tickets`: cause → link to its own ticket. A cause with no link shows as not filed yet.
  let { ticket, tickets = {} }: { ticket: number; tickets?: Partial<Record<CauseGroup['cause'], string>> } = $props();

  const rows = $derived(reports as Report[]);
  const merged = $derived(uniqueById(rows).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf, notes.oneDayMove));
  const count = (rs: Report[], key: (r: Report) => string) =>
    Object.entries(rs.reduce<Record<string, number>>((n, r) => ({ ...n, [key(r)]: (n[key(r)] ?? 0) + 1 }), {}))
      .sort((a, b) => b[1] - a[1])
      .map(([k, n]) => `${k} ×${n}`)
      .join(' · ');
  const label = (href: string) => `#${href.split('/').pop()}`;
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket}
  ({rows.length} rows in the export, {rows.length - uniqueById(rows).length} of them repeats) ·
  <strong>{groups.length}</strong>
  {groups.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>cause</th><th>reports</th><th>channels</th><th>tagged as</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.cause)}
      <tr class:unfiled={!tickets[g.cause]}>
        <td><code>{g.cause}</code></td>
        <td><strong>{g.reports.length}</strong></td>
        <td class="tags">{count(g.reports, (r) => r.channel)}</td>
        <td class="tags">{count(g.reports, (r) => r.meta.intakeTag)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>
          {#if tickets[g.cause]}<a href={tickets[g.cause]}>{label(tickets[g.cause] ?? '')}</a>{:else}not filed yet{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
  .unfiled td { background: var(--warn-bg); }
  .unfiled td:last-child { color: var(--warn); font-weight: 600; }
</style>
