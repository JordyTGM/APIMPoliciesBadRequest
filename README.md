# APIMPoliciesBadRequest AZURE APIM
Configuracion de APIM para control y redireccion de la respuesta bad request de la petición a otro servicio.
TEXTO XML:
- Policies are applied in the order they appear.
- Position <base/> inside a section to inherit policies from the outer scope.
- Comments within policies are not preserved.
- Add policies as children to the <inbound>, <outbound>, <backend>, and <on-error> elements

<policies>
    <!-- Throttle, authorize, validate, cache, or transform the requests -->
    <inbound>
        <!--<set-variable name="requestBody" value="@("{\"URL\": \"" + context.Request.Url + "\", \"METHOD\": \"" + context.Request.Method + "\"}")" /> -->
        <choose>
            <when condition="@(!string.IsNullOrEmpty(context.Request.Body?.As<string>(true)))">
                <set-variable name="requestBody" value="@((context.Request.Body.As<string>(true)))" />
            </when>
            <otherwise>
                <set-variable name="requestBody" value="&quot;&quot;" />
            </otherwise>
        </choose>
        <set-variable name="requestUrl" value="@((context.Request.Url.ToString()))" />
        <set-variable name="requestMethod" value="@((context.Request.Method.ToString()))" />
        <set-variable name="requestNameService" value="@((context.Api.Name.ToString()))" />
        <set-variable name="cleanPath" value="@((context.Request.Url.Path.Split('?')[0].Trim('/')))" />
        <set-variable name="totalSegments" value="@((Convert.ToInt32(Convert.ToString(context.Variables["cleanPath"]).Split('/').Length)))" />
        <choose>
            <when condition="@((int)context.Variables["totalSegments"] > 1)">
                <set-variable name="service" value="@((Convert.ToString(context.Variables["cleanPath"]).Split('/')[Convert.ToInt32(context.Variables["totalSegments"]) - 2]))" />
                <set-variable name="method" value="@((Convert.ToString(context.Variables["cleanPath"]).Split('/')[Convert.ToInt32(context.Variables["totalSegments"]) - 1]))" />
            </when>
            <otherwise>
                <set-variable name="service" value="Unknown" />
                <set-variable name="method" value="@(Convert.ToString(context.Variables["cleanPath"]))" />
            </otherwise>
        </choose>
        <!-- <set-variable name="method" value="@(context.Request.Url.Path.Split('/')[5])" /> -->
        <!-- <set-variable name="service" value="@(context.Request.Url.Path.Split('/')[4])" /> -->
        <!-- <set-variable name="apiGroup" value="@((context.Request.Url.Path.Trim('/').Split('/')[0]))" />-->
        <!-- <set-variable name="requestEndpoint" value="@("{\"QueryString\": \"" + context.Request.Url.QueryString + "\", \"Path\": \"" + context.Request.Url.Path + "\"}")" />-->
        <!-- <set-variable name="requestEndpoint" value="@("{"+ "\"Method\": \"" + context.Request.Method + "\","+ "\"Url\": \"" + context.Request.Url.ToString() + "\","+ "\"Headers\": " + json(context.Request.Headers) + ","+ "\"Query\": " + json(context.Request.Url.Query) + ","+ "\"Protocol\": \"" + context.Request.Protocol + "\","+ "\"Scheme\": \"" + context.Request.Url.Scheme + "\","+ "\"Host\": \"" + context.Request.Url.Host + "\","+ "\"Path\": \"" + context.Request.Url.Path + "\","+ "\"Content-Length\": " + (context.Request.ContentLength?.ToString() ?? "0") + ""+ "}")" />-->
        <base />
    </inbound>
    <!-- Control if and how the requests are forwarded to services  -->
    <backend>
        <base />
    </backend>
    <!-- Customize the responses -->
    <outbound>
        <choose>
            <when condition="@(!string.IsNullOrEmpty(context.Response.Body?.As<string>(true)))">
                <set-variable name="responseBody" value="@((context.Response.Body?.As<string>(true)))" />
            </when>
            <when condition="@((int)context.Response.StatusCode == 401)">
                <set-variable name="responseBody" value="@("{\"error\": \"Server Error\", \"status\": " + context.Response.StatusCode + ", \"message\": \"Unauthorized.\"}")" />
            </when>
            <otherwise>
                <set-variable name="responseBody" value="@("{\"error\": \"Internal Server Error\", \"status\": " + context.Response.StatusCode + ", \"message\": \"An unexpected error occurred on the server.\"}")" />
            </otherwise>
        </choose>
        <choose>
            <when condition="@((int)context.Response.StatusCode != 200 && (int)context.Response.StatusCode != 400)">
                <send-request mode="new" response-variable-name="RegistrarLog" timeout="60" ignore-error="true">
                    <set-url>http://10.16.10.10/api/v1/Log/RegistrarCabecera</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-header name="Ocp-Apim-Subscription-Key" exists-action="override">
                        <value>0e461d0c5d164f6eb3f54dd90c96182c</value>
                    </set-header>
                    <set-body template="liquid">
                        {
                            "servicio": "{{ context.Variables.service | json_escape }}",
                            "metodo":  "{{ context.Variables.method | json_escape }}",
                            "codcanal": "CRM",
                            "estado" : "Error",
                            "dato_entrada": {
                                "METHOD": "{{ context.Variables.requestMethod | json_escape }}",
                                "URL": "{{ context.Variables.requestUrl | json_escape }}",
                                "servicio": "{{ context.Variables.service | json_escape }}",
                                "metodo":  "{{ context.Variables.method | json_escape }}",
                                "BODY": {{ context.Variables.requestBody | json_escape }}
                               
                            },
                            "dato_salida": {{ context.Variables.responseBody | json_escape }}
                        }
                    </set-body>
                </send-request>
            </when>
        </choose>
        <base />
    </outbound>
    <!-- Handle exceptions and customize error responses  -->
    <on-error>
        <base />
    </on-error>
</policies>
