server_port: [2641, 2642]  # As a comma-separated list
# or
server_port:                # As a hyphenated list
  - 2641
  - 2642



vars:
  server_port_str: "{{ server_port | join(',') }}"


IFS=',' read -r -a ports <<< "${SERVER_PORT}"
for port in "${ports[@]}"; do
  # Call the startinstance.sh script for each port
done



cm_src_dest: "path_to_template1:destination_path_logback_2641.xml,path_to_template2:destination_path_logback_2642.xml"


for i in "${!ports[@]}"; do
  src="path_to_template${i}"
  dst="destination_path_logback_${ports[i]}.xml"
  # Use sed to replace any placeholders in the src file
  # Then copy the src to the dst
done



"{{ ServiceConfig.server_port if ServiceConfig.server_port is string else ServiceConfig.server_port | join(',') }}"

