# Execution plan visulization

> Python3.7 and Pycairo required



## Configure:

1. Python3.7

2. Pycairo

   > Available from https://download.lfd.uci.edu/pythonlibs/j1ulh5xc/pycairo-1.16.3-cp37-cp37m-win32.whl

   1. Get pip
   2. pip install whell
   3. pip install *.whl

## Use:

* A sample is provided. Try it by: (`explain_plan_no_shard.py` is for stand-alone node)

  1. Delete `shard_sample.svg, shard_sample.png`
  2. Run`python execution_plan_shard_sample.py`

* The entrance is`execution_plan_shard.py`.Run as follows:

  ```shell
  python execuetion_plan_shard.py "set_1530356281_3" "{\"query_block\": {\"select_id\": 1,\"cost_info\": {\"query_cost\": \"0.45\"},\"table\": {\"table_name\": \"t\",\"access_type\": \"ALL\",\"rows_examined_per_scan\": 2,\"rows_produced_per_join\": 2,\"filtered\": \"100.00\",\"cost_info\": {\"read_cost\": \"0.25\",\"eval_cost\": \"0.20\",\"prefix_cost\": \"0.45\",\"data_read_per_join\": \"32\"},\"used_columns\": [\"a\",\"b\"]}}}" "set_1530356281_1" "{\"query_block\": {\"select_id\": 1,\"cost_info\": {\"query_cost\": \"2.40\"},\"nested_loop\": [{\"table\": {\"table_name\": \"t\",\"access_type\": \"ALL\",\"rows_examined_per_scan\": 1,\"rows_produced_per_join\": 1,\"filtered\": \"100.00\",\"cost_info\": {\"read_cost\": \"1.00\",\"eval_cost\": \"0.20\",\"prefix_cost\": \"1.20\",\"data_read_per_join\": \"16\"},\"used_columns\": [\"a\",\"b\"]}},{\"table\": {\"table_name\": \"t2\",\"access_type\": \"ALL\",\"rows_examined_per_scan\": 1,\"rows_produced_per_join\": 1,\"filtered\": \"100.00\",\"using_join_buffer\": \"Block Nested Loop\",\"cost_info\": {\"read_cost\": \"1.00\",\"eval_cost\": \"0.20\",\"prefix_cost\": \"2.40\",\"data_read_per_join\": \"16\"},\"used_columns\": [\"a\"],\"attached_condition\": \"(`test`.`t2`.`a` = `test`.`t`.`a`)\"}}]}}"
  ```

  **Args explanation:**

  - "set_1530356281_3": Dataset ID in format string.
  - "{\"query_block\": ... \"b\"]}}}": Execution plan in set_1530356281_3, in format sitrng. 

  
