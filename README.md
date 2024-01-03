if(deployType == 'EC2')
{
serviceStructure = "<hostname>||<service-name>:<artifact>|<service-name>:<artifact>"
}
if(deployType == 'EC2-AEM')
{
serviceStructure = "<both>||<service-name>:<artifact>|<service-name>:<artifact>"

}
stage_param_map = [
  "service_Deploy": [
  ["parameters": "serviceList", "param_value": serviceStructure ],
  ],
"service_Restart": [
      ["parameters": "serviceList", "param_value": "<hostname>||<service-name>,<service-name>" ],
  ],
"service_dispatcher":[
     ["parameters": "dispatcherPath", "param_value": "<path>" ],
],
"config_Deploy":[
     ["parameters": "config_Deploy", "param_value": "true" ],
]
]
def stageList = stages.split(",")
html_to_be_rendered = "<table><tr>"
stageList.each{ stg ->
	param_list = stage_param_map[stg]
	param_list.each { p ->
		html_to_be_rendered = """
			${html_to_be_rendered}
			<tr>
			<td>
				<input type=\"hidden\" class=\" \" name=\"value\" value=\"${p.parameters}\">
			<label title=\"${p.parameters}\" class=\" \">${p.parameters}</label>
			</td>
			<td style=\"width:100%\">
			<input type=\"text\" style=\"width:60%;\" name=\"value\" value=\"${p.param_value}\"> </br>
			</td>
			</tr>
		"""
    }
}


html_to_be_rendered = "${html_to_be_rendered}</tr></table>"

return html_to_be_rendered


