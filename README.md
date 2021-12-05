target="srv2"
type="$target-bp28m-before"
data_stream="/usr/share/xml/scap/ssg/content/ssg-rhel7-ds-1.2.xml"
profile="xccdf_org.ssgproject.content_profile_anssi_nt28_minimal"
cpe_dict="/usr/share/xml/scap/ssg/content/ssg-rhel7-cpe-dictionary.xml"
ssh-copy-id $target

ssh $target "yum -y install scap-security-guide"

//installation  station de controle
curl -L https://git.io/JMiO5 | bash -x

//scanner
type="$target-bpm28-before"
profile="xccdf_org.ssgproject.content_profile_anssi_nt28_minimal"
oscap-ssh --sudo root@$target 22 xccdf eval \
--fetch-remote-resource \
--profile $profile \
--results $type-results.xml \
--report $type-report.html \
--oval-results \
--cpe $cpe_dict \
$data_stream

//vérification des fichiers générés
ls -l srv2*
ls -l ssg*

//géneration du guide 
oscap xccdf generate guide \
--profile $profile \
--output $type-guide.html \
$type-results.xml

//lancement du service web 
python3 -m http.server 8080

//remédiation: Scan d'une règle
rule1="xccdf_org.ssgproject.content_rule_accounts_maximum_age_login_defs"
type="rule1-$target-bpm28-before"
profile="xccdf_org.ssgproject.content_profile_anssi_nt28_minimal"
oscap-ssh --sudo root@$target 22 xccdf eval \
--fetch-remote-resource \
--profile $profile \
--results $type-results.xml \
--report $type-report.html \
--oval-results \
--cpe $cpe_dict \
--rule $rule1 \
$data_stream

//géhérer une rémédiation
result_id=$(oscap info $type-results.xml | grep 'Result ID' | sed 's/[[:blank:]]Result ID: //')
oscap xccdf generate fix \
--fix-type ansible \
--output $type-playbook.yml \
--profile $profile \
--result-id $result_id \
$type-results.xml

//appliquer la remédiation
ansible-playbook -i "$target," $type-playbook.yml

//relancer l'audit
rule1="xccdf_org.ssgproject.content_rule_accounts_maximum_age_login_defs"
type="rule1-$target-bpm28-after"
profile="xccdf_org.ssgproject.content_profile_anssi_nt28_minimal"
oscap-ssh --sudo root@$target 22 xccdf eval \
--fetch-remote-resource \
--profile $profile \
--results $type-results.xml \
--report $type-report.html \
--oval-results \
--cpe $cpe_dict \
--rule $rule1 \
$data_stream
