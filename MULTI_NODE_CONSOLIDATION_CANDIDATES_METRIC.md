# Multi-Node Consolidation Candidates Metric

## Overview
The `karpenter_voluntary_disruption_multi_node_consolidation_candidates` metric tracks the number of candidates that are passed into the multi-node consolidation algorithm after budget filtering.

## Metric Details

**Name:** `karpenter_voluntary_disruption_multi_node_consolidation_candidates`

**Type:** Histogram

**Labels:** None

**Help Text:** "Alpha metric prone to change. Number of candidates passed to multi-node consolidation after budget filtering."

**Buckets:** `[2, 5, 10, 20, 30, 50, 75, 100]`

## What It Measures

This metric captures the count of disruptable candidates that remain after:
1. Sorting candidates by disruption order
2. Filtering out nodes that would violate disruption budgets
3. Filtering out empty nodes (nodes with no reschedulable pods)

The recorded value represents the input size to the `firstNConsolidationOption` method, which uses a binary search algorithm to find the optimal batch of nodes to consolidate together.

## Why Use a Histogram?

A histogram was chosen over a gauge because:
- **Distribution Analysis**: Shows the distribution of candidate counts across multiple consolidation attempts
- **Pattern Recognition**: Helps identify if consolidation typically operates on small batches (2-5 nodes) vs large batches (50+ nodes)
- **Percentile Queries**: Enables p50, p95, p99 analysis to understand typical vs exceptional cases
- **Event-Driven Nature**: This value changes discretely at consolidation events rather than continuously

## Example Queries

### Average Number of Candidates Per Consolidation Attempt
```promql
rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_sum[5m]) / 
rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_count[5m])
```

### Distribution of Candidate Counts (Histogram)
```promql
histogram_quantile(0.50, rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_bucket[5m])) # p50
histogram_quantile(0.95, rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_bucket[5m])) # p95
histogram_quantile(0.99, rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_bucket[5m])) # p99
```

### Consolidation Attempts by Candidate Count Range
```promql
rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_bucket[5m])
```

### Percentage of Attempts with High Candidate Counts (50+)
```promql
(
  rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_bucket{le="100"}[5m]) -
  rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_bucket{le="50"}[5m])
) / rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_count[5m]) * 100
```

## Use Cases

### 1. Capacity Planning
Understand typical batch sizes to predict consolidation behavior and plan for compute capacity during consolidation windows.

### 2. Budget Effectiveness
If candidate counts are consistently low, it may indicate that disruption budgets are too restrictive.

### 3. Performance Monitoring
Large candidate counts may correlate with longer consolidation times. Cross-reference with:
- `karpenter_voluntary_disruption_multi_node_consolidation_batch_iterations`
- `karpenter_voluntary_disruption_multi_consolidation_failed_sim_duration_seconds`

### 4. Cluster Health
Sudden changes in candidate distribution may indicate:
- Changes in workload patterns
- NodePool configuration changes
- Budget policy adjustments

## Related Metrics

Compare with these other multi-node consolidation metrics:

- **`karpenter_voluntary_disruption_multi_node_consolidation_batch_size`**: Actual number of nodes in successful consolidations (what was consolidated)
- **`karpenter_voluntary_disruption_multi_node_consolidation_batch_iterations`**: Number of binary search iterations taken
- **`karpenter_voluntary_disruption_multi_consolidation_failed_sim_duration_seconds`**: Time spent on failed iterations
- **`karpenter_voluntary_disruption_consolidation_timeouts_total`**: Number of timeouts (may correlate with high candidate counts)

## Example Alert

```yaml
- alert: HighMultiNodeConsolidationCandidates
  expr: |
    histogram_quantile(0.95, 
      rate(karpenter_voluntary_disruption_multi_node_consolidation_candidates_bucket[10m])
    ) > 80
  for: 30m
  annotations:
    summary: "High number of multi-node consolidation candidates"
    description: "95th percentile of consolidation candidates is {{ $value }}, indicating large batch sizes that may impact consolidation performance"
```

## Notes

- **Alpha Metric**: This metric is marked as "prone to change" and may be modified or removed in future versions
- **Measurement Point**: Recorded before the binary search algorithm runs, not after
- **Zero Values**: If no candidates pass budget filtering, the metric will record 0
- **Max Clamping**: The actual consolidation is clamped to max 100 nodes, but this metric records the pre-clamped value
