<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Key = 'parser' | 'hint' | 'unplaced' | 'shift';
  let { ticket, tickets, labels }: { ticket: number; tickets: Partial<Record<Key, string>>; labels: Partial<Record<Key, string>> } = $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  const groups = $derived(groupByCause(merged, tenantOf));
  const footnotes = $derived(groups.map((g) => ({ cause: g.cause, moved: readAs(g, notes.oneDayMove) })).filter((f) => f.moved > 0));
  const problems = $derived(groups.length + (footnotes.length > 0 ? 1 : 0));
  const list = (n: Record<string, number>) =>
    Object.entries(n)
      .sort((a, b) => b[1] - a[1])
      .map(([k, v]) => `${k} ×${v}`)
      .join(' · ');
  const channels = (rs: Report[]) => list(rs.reduce<Record<string, number>>((n, r) => ({ ...n, [r.channel]: (n[r.channel] ?? 0) + 1 }), {}));
  const issue = (url?: string) => url?.replace(/^.*\/issues\//, '#');
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ({rows.length} export rows, {rows.length - merged.length} repeated) ·
  <strong>{problems}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>came in by</th><th>intake tagged</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.cause)}
      <tr>
        <td><code>{g.cause}</code><br /><span class="label">{labels[g.cause] ?? ''}</span></td>
        <td><strong>{g.reports.length}</strong></td>
        <td>{g.tenants}</td>
        <td class="small">{channels(g.reports)}</td>
        <td class="small">{list(g.taggedAs)}</td>
        <td class="small">{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if tickets[g.cause]}<a href={tickets[g.cause]}>{issue(tickets[g.cause])}</a>{:else}none{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#each footnotes as f (f.cause)}
  <p class="note">
    <strong>{f.moved}</strong> of the <code>{f.cause}</code> reports read as a one-day move, not a swap: <code>shift</code>,
    {labels.shift ?? ''}. Counted under <code>{f.cause}</code>, where the desk files them, and ticketed on their own as
    {#if tickets.shift}<a href={tickets.shift}>{issue(tickets.shift)}</a>{:else}none{/if}.
  </p>
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .small, .label, .note { color: var(--muted); font-size: 0.85em; }
</style>
