## Arquitetura
Vamos fazer deploy de um moodelo de Machine Learning criado com TensorFlow usando o TensorFlow Serving, Docker e Kubernetes com autoscale de pods. A implantação pode ser acessada por terminais externos (ou seja, seus usuários) por meio de um serviço exposto. Isso traz solicitações de inferência para os servidores de modelo e responde com previsões de seu modelo. A implantação aumentará ou diminuirá os pods com base na utilização da CPU. Ele começará com um pod, mas quando a carga exceder um ponto predefinido, ele acionará pods adicionais para compartilhar a carga.

## Start Minikube
Precisamos mapear uma pasta da nossa máquina fisica para o cluster Minikube.
```
minikube start --mount=True --mount-string="D:/formacao_mlops/meus/intro_kubernetes/C4_W2_Lab_2_Intro_to_Kubernetes:/var/tmp"
```

## YAML Files

### Config Maps
Vamos criar um mapa de configuração que define uma variável MODEL_NAME e MODEL_PATH. Essas duas variáveis são usadas quando o container está sendo inicializado e inicia o servidor de modelo (Tensor Flow Serving) e usa as variáveis de ambiente MODEL_BASE_PATH e MODEL_NAME para encontrar o modelo.
```
kubectl apply -f yaml/configmap.yaml
kubectl describe cm tfserving-configs
```

### Deployment
Ele inicia uma réplica, usa a imagem "tensorflow/serving" como a imagem do contêiner e define as variáveis de ambiente por meio da tag envFrom. Ele também expõe a porta 8501 do contêiner porque nos enviaremos solicitações HTTP a ele posteriormente. Ele também define limites de CPU e memória e monta o volume da VM do Minikube para o contêiner.
```
kubectl apply -f yaml/deployment.yaml
kubectl get deploy
```

### Expondo o deployment atráves de um service
Precisamos criar um serviço para que o aplicativo possa ser acessado fora do cluster. Ele define um serviço NodePort que expõe a porta 30001 do nó. As solicitações enviadas para esta porta serão enviadas para a targetPort especificada dos contêineres, que é 8501.
```
kubectl apply -f yaml/service.yaml
kubectl get svc tf-serving-service
```

## Teste
Podemos tentar acessar a implantação agora como uma verificação de integridade. Executar o comando "minikube ip" primeiro para obter o endereço IP do nó do Minikube. Nosso caso é: 127.0.0.1. Mas como estamos usando Docker e a rede ainda não foi configurada, esse IP ai não vai funcionar. Então usamos o comando "minikube service tf-serving-service". Isso abre um túnel para o seu serviço com uma porta aleatória. Pegue o URL na caixa inferior direita e use-o no comando curl. O seguinte comando curl enviará uma linha de solicitações de inferência ao serviço Nodeport (Link para formato e Windows: https://mkyong.com/web/curl-post-json-data-on-windows/).
```
curl.exe -d "{\"instances\": [1.0, 2.0, 5.0]}" -X POST http://127.0.0.1:52898/v1/models/half_plus_two:predict
```

## Autoscaler
O Kubernetes fornece um pod autoscaler horizontal (HPA) para criar ou remover réplicas com base em métricas observadas. Para fazer isso, o HPA consulta um Metrics Server para medir a utilização de recursos, como CPU e memória. O Metrics Server não é iniciado por padrão no Minikube e precisa ser ativado com o seguinte comando:
```
minikube addons enable metrics-server
```

Isso inicia uma implantação de servidor de métricas no namespace do sistema kube. Execute o comando abaixo e aguarde a implantação estar pronta.
```
kubectl get deployment metrics-server -n kube-system
```

Agora podemos criar nosso autoscaler:
```
kubectl apply -f yaml/autoscale.yaml
kubectl get hpa
```

## Stress Test
Para testar a capacidade de escalonamento automático da implantação, usamos o script .bat. Basta executá-lo no seu terminal.
Existem várias maneiras de monitorar o comportameto do ambiente nesse teste. A mais fácil é usar o painel embutido do Minikube. Você pode iniciá-lo executando:
```
minikube dashboard
```

## Deletar o Ambiente
Para deletera todos os recursos que criamos, basta executar (vai deletar apenas os recursos dos arquivos YAML desse diretorio):
```
kubectl delete -f yaml
```

Para criar tudo de uma vez só:
```
kubectl apply -f yaml
```
