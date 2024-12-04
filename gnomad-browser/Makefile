demo_name = $(error Please set demo_name flag)
docker_tag = $(error Please set docker_tag flag)

demo:
	cp -r example-demo $(demo_name)
	sed -i '' 's/PLACEHOLDER/$(demo_name)/g' $(demo_name)/kustomization.yaml
	sed -i '' 's/DOCKERTAG/$(docker_tag)/g' $(demo_name)/kustomization.yaml

clean:
	rm -rf ./$(demo_name)
