<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in the ticket, each with the ticket it was split into. Causes are counted the
  // desk's way (CauseTable); the reports the desk read as a one-day move (desk-notes.json) get a row of
  // their own, `shift`, and stay counted where they are filed.
  type Row = CauseGroup['cause'] | 'shift';
  // `tickets`: problem → link to its own ticket. A problem with no link shows as not filed yet.
  let { ticket, tickets = {} }: { ticket: number; tickets?: Partial<Record<Row, string>> } = $props();

  const rows = $derived(reports as Report[]);
  const merged = $derived(uniqueById(rows).filter((r) => r.mergedInto === ticket));
  const causes = $derived(groupByCause(merged, tenantOf));
  const shifted = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const groups = $derived([
    ...causes.map((g) => ({ ...g, row: g.cause as Row, alsoIn: null as string | null })),
    ...(shifted.length
      ? groupByCause(shifted, () => undefined).map((g) => ({
          ...g,
          row: 'shift' as Row,
          alsoIn: causes.filter((c) => readAs(c, notes.oneDayMove)).map((c) => c.cause).join(', '),
        }))
      : []),
  ]);
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
    {#each groups as g (g.row)}
      <tr class:unfiled={!tickets[g.row]}>
        <td><code>{g.row}</code></td>
        <td><strong>{g.reports.length}</strong>{#if g.alsoIn}<span class="tags"> also counted under <code>{g.alsoIn}</code></span>{/if}</td>
        <td class="tags">{count(g.reports, (r) => r.channel)}</td>
        <td class="tags">{count(g.reports, (r) => r.meta.intakeTag)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>
          {#if tickets[g.row]}<a href={tickets[g.row]}>{label(tickets[g.row] ?? '')}</a>{:else}not filed yet{/if}
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
