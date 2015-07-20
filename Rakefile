require 'aws-sdk-v1'

SUCCESS_STATS = [:create_complete, :update_complete, :update_rollback_complete]
FAILED_STATS = [:create_failed, :update_failed]
DEFAULT_RECIPES = [
  'apt',
  'docker::install',
  'elasticsearch',
  'elasticsearch::aws',
  'elasticsearch::proxy',
  'elasticsearch::plugins',
  'java',
  'layer-custom::esmonit',
  'layer-custom::cloudwatch-custom'
].join(',')

def opsworks
  AWS::OpsWorks::Client.new(region: 'us-east-1')
end

def wait_for_cf_stack_op_to_finish(stack)
  stats = stack.status.downcase.to_sym
  puts "[Stack: #{stack.name}]: current status: #{stats}"

  until SUCCESS_STATS.include?(stats)
    sleep 15
    stats = stack.status.downcase.to_sym
    fail 'Resource stack update failed!' if FAILED_STATS.include?(stats)
    puts "[Stack: #{stack.name}]: current status: #{stats}"
  end
end

def cf_query_output(stack, key)
  output = stack.outputs.find { |o| o.key == key }
  output && output.value
end

def instance_online?(instance_id)
  response = opsworks.describe_instances(instance_ids: [instance_id])
  response[:instances].first[:status] == 'online'
end

def instance_status(instance_id)
  begin
    response = opsworks.describe_instances(instance_ids: [instance_id])
  rescue AWS::OpsWorks::Errors::ResourceNotFoundException
    return 'nonexistent'
  end
  response[:instances].first[:status].tap do |status|
    fail "Instance #{instance_id} has a failed status #{status}" if status =~ /fail|error/i
  end
end

def wait_for_instance(instance_id, status)
  while (ins_status = instance_status(instance_id)) != status
    puts "[Instance #{instance_id}] waiting for instance to become #{status}. Current status: #{ins_status}"
    sleep 10
  end
end

def deployment_status(deployment_id)
  begin
    response = opsworks.describe_deployments(deployment_ids: [deployment_id])
  rescue AWS::OpsWorks::Errors::ResourceNotFoundException
    return 'nonexistent'
  end
  response[:deployments].first[:status].tap do |status|
    fail "Deployment #{deployment_id} has a failed status #{status}" if status =~ /fail|error/i
  end
end

def wait_for_deployment(deployment_id, status)
  while (dep_status = deployment_status(deployment_id)) != status
    puts "[Deployment #{deployment_id} waiting for deployment to become #{status}. Current status: #{dep_status}"
    sleep 10
  end
end

def all_availability_zones
  ec2 = AWS::EC2.new
  ec2.availability_zones.map(&:name)
  ['eu-west-1a', 'eu-west-1b']
end

def get_subnet(az)
  ec2 = AWS::EC2.new
  response = ec2.subnets
  response.filter('availability-zone', az).filter('subnet-id', ['subnet-49ef3a10', 'subnet-a1c44fc4']).first.subnet_id
end

def get_all_instances(layer_id)
  response = opsworks.describe_instances(layer_id: layer_id)
  response[:instances]
end

def attach_ebs_volumes(instance_id, volume_ids)
  volume_ids.each do |volume_id|
    puts "Attaching EBS volume #{volume_id} to instance #{instance_id}"
    opsworks.assign_volume(volume_id: volume_id, instance_id: instance_id)
  end
end

def detach_ebs_volumes(instance_id)
  response = opsworks.describe_volumes(instance_id: instance_id)
  volume_ids = response[:volumes].map { |v| v[:volume_id] }
  volume_ids.each do |volume_id|
    puts "Detaching EBS volume #{volume_id} from instance #{instance_id}"
    opsworks.unassign_volume(volume_id: volume_id)
  end

  volume_ids
end

def create_instance(stack_id, layer_id, az)
  subnet = get_subnet(az)
  opsworks.create_instance(stack_id: stack_id,
                           layer_ids: [layer_id],
                           instance_type: ENV['INSTANCE_TYPE'] || 'c3.large',
                           install_updates_on_boot: !ENV['SKIP_INSTANCE_PACKAGE_UPDATES'],
                           subnet_id: subnet,
                           availability_zone: az)
end

def update_instances(stack_id, layer_id, count)
  azs = all_availability_zones
  existing_instances = get_all_instances(layer_id)
  count_to_create = count - existing_instances.size
  new_instances = (1..count_to_create).map do |i|
    instance = create_instance(stack_id, layer_id, azs[(existing_instances.size + i) % azs.size])
    puts "Created instance, id: #{instance[:instance_id]}, starting the instance now."
    opsworks.start_instance(instance_id: instance[:instance_id])
    instance
  end

  new_instances.each do |instance|
    wait_for_instance(instance[:instance_id], 'online')
  end

  if ENV['REPLACE_INSTANCES'] != 'true'
    existing_instances.each do |instance|
      puts "Updating custom cookbooks on instance #{instance[:hostname]}, id: #{instance[:instance_id]}"
      deployment = opsworks.create_deployment(stack_id: stack_id,
                                              instance_ids: [instance[:instance_id]],
                                              command: { name: 'update_custom_cookbooks' })
      wait_for_deployment(deployment[:deployment_id], 'successful')

      puts "Running setup command on instance #{instance[:hostname]}, id: #{instance[:instance_id]}"
      deployment = opsworks.create_deployment(stack_id: stack_id,
                                              instance_ids: [instance[:instance_id]],
                                              command: { name: 'setup' })
      wait_for_deployment(deployment[:deployment_id], 'successful')
    end
  else
    replace_instances(existing_instances)
  end
end

def replace_instances(existing_instances)
  puts 'Replacing existing instances..' if existing_instances.size > 0

  existing_instances.each do |instance|
    puts "Stopping instance #{instance[:hostname]}, id: #{instance[:instance_id]}"
    opsworks.stop_instance(instance_id: instance[:instance_id])
    wait_for_instance(instance[:instance_id], 'stopped')
    ebs_volume_ids = detach_ebs_volumes(instance[:instance_id])

    puts 'Creating replacement instance'
    replacement = create_instance(stack_id, layer_id, instance[:availability_zone])
    attach_ebs_volumes(replacement[:instance_id], ebs_volume_ids)

    puts "Starting new instance, id: #{replacement[:instance_id]}"
    opsworks.start_instance(instance_id: replacement[:instance_id])
    wait_for_instance(replacement[:instance_id], 'online')

    puts "Deleting old instance #{instance[:hostname]}, #{instance[:instance_id]}"
    opsworks.delete_instance(instance_id: instance[:instance_id])
  end
end

def min_master_node_count(instance_count)
  instance_count <= 2 ? 1 : (instance_count / 2 + 1)
end

def environment
  ENV['ENVIRONMENT'] || 'my'
end

def stack_name
  "#{environment}-es"
end

def route53_zone_name
  ENV['ROUTE53_ZONE_NAME'] || 'appdev.io'
end

def get_required(name)
  ENV[name] || fail("You must provide the environment variable #{name}")
end

def add_param_if_set(params, param_name, env_var)
  return unless ENV[env_var]
  puts "Setting CloudFormation param #{param_name.inspect} => #{ENV[env_var].inspect}"
  params.merge!(param_name => ENV[env_var])
end

desc 'Provisions the ElasticSearch cluster'
task :provision do
  cfm = AWS::CloudFormation.new
  instance_count = (ENV['INSTANCE_COUNT'] || '2').to_i
  template = File.read('opsworks-service.template')
  cf_stack = cfm.stacks[stack_name]

  params = {
    'SSLCertificateName' => get_required('SSL_CERTIFICATE_NAME'),
    'Route53ZoneName' => route53_zone_name,
    'SearchDomainName' => ENV['SEARCH_DOMAIN_NAME'] || "#{stack_name}.#{route53_zone_name}",

    'SshKeyName' => ENV['SSH_KEY_NAME'] || 'elasticsearch',
    'SearchUser' => ENV['SEARCH_USER'] || 'elasticsearch',
    'SearchPassword' => ENV['SEARCH_PASSWORD'] || 'pass',
    'InstanceCount' => instance_count.to_s,
    'MinMasterNodes' => min_master_node_count(instance_count).to_s,
    'ClusterName' => stack_name,
    'RecipeList' => DEFAULT_RECIPES
  }

  add_param_if_set(params, 'ElasticSearchVersion', 'ELASTICSEARCH_VERSION')
  add_param_if_set(params, 'ElasticSearchAWSCloudPluginVersion', 'ELASTICSEARCH_AWS_PLUGIN_VERSION')

  if cf_stack.exists?
    begin
      puts "Updating CloudFormation stack #{stack_name}"
      cf_stack.update(template: template, parameters: params)
    rescue => e
      raise unless e.message =~ /No updates are to be performed/
      puts 'Your CloudFormation stack is already up to date'
    end
  else
    puts "Creating CloudFormation stack #{stack_name}"
    cf_stack = cfm.stacks.create(stack_name, template, parameters: params)
  end

  wait_for_cf_stack_op_to_finish(cf_stack)

  unless ENV['SKIP_INSTANCE_UPDATE'] == 'true'
    stack_id = cf_query_output(cf_stack, 'StackId')
    layer_id = cf_query_output(cf_stack, 'LayerId')

    update_instances(stack_id, layer_id, instance_count)
  end
end

desc 'Destroys the ElasticSearch cluster'
task :destroy do
  cfm = AWS::CloudFormation.new
  cf_stack = cfm.stacks[stack_name]
  if cf_stack.exists?
    puts "Destroying environment #{environment}"

    layer_id = cf_query_output(cf_stack, 'LayerId')

    get_all_instances(layer_id).each do |instance|
      puts "Stopping instance #{instance[:hostname]}"
      opsworks.stop_instance(instance_id: instance[:instance_id])
      wait_for_instance(instance[:instance_id], 'stopped')

      puts "Deleting instance #{instance[:hostname]}"
      opsworks.delete_instance(instance_id: instance[:instance_id])
      wait_for_instance(instance[:instance_id], 'nonexistent')
    end

    puts "Deleting OpsWorks stack #{stack_name}"
    cf_stack.delete
  else
    puts "Environment #{environment} does not exist"
  end
end
