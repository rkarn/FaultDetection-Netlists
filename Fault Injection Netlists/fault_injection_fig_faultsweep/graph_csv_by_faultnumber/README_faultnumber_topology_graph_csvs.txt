Fault-number-sweep topology-aware graph CSVs.

Input root:
  fault_injection_fig_faultsweep

Clean netlist directory used:
  fault_injection_fig_faultsweep/clean_netlists

Output directory:
  fault_injection_fig_faultsweep/graph_csv_by_faultnumber

Generated range-specific CSV pairs:
  nodes_faultnumber_100_500.csv
  edges_faultnumber_100_500.csv
  nodes_faultnumber_500_1000.csv
  edges_faultnumber_500_1000.csv
  nodes_faultnumber_1000_2000.csv
  edges_faultnumber_1000_2000.csv

Fault topology embedding:
  stuck_at:
    Each injected stuck-at assignment creates a pseudo source node:
      SA0_<net> or SA1_<net>
    The pseudo node overrides the original driver of the selected net.

  glitch:
    Each injected glitch creates a pseudo source node:
      GL_<net>
    The pseudo node overrides the original driver of the selected net.

  bridging:
    Each injected bridge creates a pseudo node:
      BR_<net1>_<net2>
    The pseudo node is added as a load of net2 and overrides the driver of net1.

Labels:
  Only injected pseudo fault nodes receive label=1.
  Original gate instances remain label=0.

Schemas:
  The nodes and edges CSV headers match the original topology-aware script:
    nodes: netlist_file,fault_type,node_id,gate_type,in_degree,out_degree,total_degree,path_depth,CC0,CC1,CO,is_input,is_output,label
    edges: netlist_file,fault_type,src_node,dst_node,fan_out

Metadata:
  fault_metadata_detailed.csv and fault_injection_summary.csv are used when available.
  If unavailable, the script falls back to parsing injected comments and tail assignments.
