lichee.jar:
	rm -rf build/; \
	mkdir build/; \
	javac -g -classpath $$(ls -1 lib/*.jar | tr "\n" ":") -sourcepath src/ -d build/ src/lineage/LineageEngine.java; \
	jar -cfe release/lichee.jar lineage.LineageEngine -C build .;