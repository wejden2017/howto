###############################
# main.tf
###############################

provider "aws" {
  region = "eu-west-1" # Spécifie la région AWS
}

# Import d'une VPC existante (doit être remplacé par ton ID de VPC)
data "aws_vpc" "main" {
  id = "vpc-xxxxxxxx"
}

# Import de deux sous-réseaux privés existants
data "aws_subnet" "private_a" {
  id = "subnet-aaaaaaa"
}

data "aws_subnet" "private_b" {
  id = "subnet-bbbbbbb"
}

# Import d'une instance EC2 privée existante qui héberge Spring Boot
# L'application doit écouter sur le port 8080

data "aws_instance" "springboot_ec2" {
  instance_id = "i-xxxxxxxxxxxxxxxxx"
}

# Création d'un Target Group pour envoyer le trafic HTTP vers le port 8080 de l'EC2
resource "aws_lb_target_group" "spring_tg" {
  name        = "spring-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = data.aws_vpc.main.id
  target_type = "instance" # On cible une instance EC2
}

# Ajout de l'EC2 au Target Group
resource "aws_lb_target_group_attachment" "spring_tg_attachment" {
  target_group_arn = aws_lb_target_group.spring_tg.arn
  target_id        = data.aws_instance.springboot_ec2.id
  port             = 8080
}

# Création d'un NLB (Network Load Balancer) interne qui distribuera le trafic dans le VPC
resource "aws_lb" "internal_nlb" {
  name               = "internal-nlb"
  internal           = true # Important : interne signifie non accessible publiquement
  load_balancer_type = "network"
  subnets            = [
    data.aws_subnet.private_a.id,
    data.aws_subnet.private_b.id
  ]
}

# Listener du NLB pour transférer le trafic TCP 8080 vers le Target Group
resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.internal_nlb.arn
  port              = 8080
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.spring_tg.arn
  }
}

# Création d'un VPC Link pour permettre à API Gateway de parler à l'intérieur du VPC
resource "aws_apigatewayv2_vpc_link" "vpc_link" {
  name        = "spring-vpc-link"
  subnet_ids  = [
    data.aws_subnet.private_a.id,
    data.aws_subnet.private_b.id
  ]
}

# Création de l'API Gateway HTTP (type REST serait aussi possible)
resource "aws_apigatewayv2_api" "spring_api" {
  name          = "spring-api"
  protocol_type = "HTTP"
}

# Intégration entre l'API Gateway et le NLB via le VPC Link
resource "aws_apigatewayv2_integration" "spring_integration" {
  api_id                 = aws_apigatewayv2_api.spring_api.id
  integration_type       = "HTTP_PROXY"
  integration_uri        = "http://${aws_lb.internal_nlb.dns_name}:8080" # Redirection vers le NLB
  connection_type        = "VPC_LINK" # Utilisation du VPC Link pour le réseau privé
  connection_id          = aws_apigatewayv2_vpc_link.vpc_link.id
  integration_method     = "ANY"
  payload_format_version = "1.0"
}

# Définition de la route dans l'API Gateway (toutes les méthodes sur /)
resource "aws_apigatewayv2_route" "spring_route" {
  api_id    = aws_apigatewayv2_api.spring_api.id
  route_key = "ANY /" # Accepte toutes les méthodes HTTP

  target = "integrations/${aws_apigatewayv2_integration.spring_integration.id}"
}

# Déploiement automatique d'un stage par défaut
resource "aws_apigatewayv2_stage" "default_stage" {
  api_id      = aws_apigatewayv2_api.spring_api.id
  name        = "$default"
  auto_deploy = true
}

# Affiche l'URL publique de l'API Gateway à la fin du déploiement
output "api_gateway_url" {
  value = aws_apigatewayv2_api.spring_api.api_endpoint
}
